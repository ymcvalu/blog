---
title: java与plsql类型映射
date: 2017-10-18 21:18:51
tags:
	- java
	- plsql
---



# java类型与数据库类型

### 场景

需要将`JAVA`中的复杂数据类型作为plsql存储过程参数传递

- 将参数转换为`json`格式的字符串：`json`的转换和解析都需要时间，影响效率，而且需要为schema导入一套`json`处理的`object`，不利于迁移
- 使用`Struct`和`Array`等，将`java`的类型映射成数据库的`object`和`Array`

### java

首先，定义`java bean`：

```java
public class TechnicalInfo {
    //定时/实时
    public String timingType;
    //全量/增量
    public String amountType;
    //同步/异步
    public String syncType;
    //传输逻辑
    public String transferLogic;
    //服务地址
    public String serviceAddress;
    //技术场景
    public String techScene;
   
}

/**
 * Fields
**/
import java.sql.Connection;
import java.util.List;
import java.util.LinkedList;
import java.lang.reflect.Field;
import oracle.sql.STRUCT;
import oracle.sql.StructDescriptor;

public class Fields {
    //字段名称
    public String name;
    //类型
    public String type;
    //长度
    public String length;
    //位置
    public String position;
    //层级
    public String level;
    //描述
    public String desc;
    //备注
    public String remarks;

    public STRUCT toStruct(Connection conn) throws Exception{
     	//使用oracle的数据库接口，类型描述符，对应一个object类型
        StructDescriptor sd = new StructDescriptor("SERVICE_FIELDS",conn);
         
        List<Object> objs = new LinkedList<>();
        Field[] fs = Fields.class.getDeclaredFields();
        for (Field f : fs){
            if (f.getType() == String.class){
                objs.add(f.get(this));
            }
        }
        //创建一个对应plsql中object对象的struct
        //objs内的数据与object中声明顺序一致，一一对应
        return new STRUCT(sd,conn,objs.toArray());
    }

}

/**
 * ServiceInfo
**/
import java.util.LinkedList;
import java.util.List;
import java.sql.Connection;
import java.lang.reflect.Field;
import java.sql.Clob;
import java.sql.Struct;
import oracle.sql.ARRAY;
import oracle.sql.ArrayDescriptor;

 public class ServiceInfo {
    //服务编号
    public String serviceNo;
    //接口名称
    public String serviceName;
    //服务版本
    public String serviceVersion;
    //服务提供方
    public String serviceProvider;
    //联系人
    public String contact;
    //一级分类
    public String AClass;
    //二级分类
    public String BClass;
    //三级分类
    public String CClass;
    //服务描述
    public String serviceDesc;
    //业务场景
    public String businessScene;
    //备注
    public String remarks;

    //触发系统报文
    public String triggeredMsg;
    //接收系统报文
    public String receiveMsg;

    //技术信息
    public TechnicalInfor technicalInfor;

    //字段
    public List<Fields> fieldses;
  
    private ServiceInfo() {
    }


    public static ServiceInfo build() {
        ServiceInfo si = new ServiceInfo();
        si.technicalInfor = new TechnicalInfor();
        si.fieldses = new LinkedList<>();
        return si;
    }

    public Struct toStruct(Connection conn) throws Exception {
      List<Object> objs = new LinkedList<>();
     
        Class clazz = this.getClass();
        Field[] fs = clazz.getDeclaredFields();
        for (Field f : fs) {
            if ("receiveMsg".equals(f.getName()) || "triggeredMsg".equals(f.getName())) {
                Clob c = conn.createClob();
                
                c.setString(1, (String)f.get(this));
                objs.add(c);
            } else if (f.getType() == String.class) {
              objs.add(f.get(this));
            }else if (f.getType() == TechnicalInfo.class){
                Field[] ffs = TechnicalInfor.class.getDeclaredFields();
                for (Field ff: ffs){
                    if (ff.getType()==String.class){
                          objs.add(ff.get(this.technicalInfor));
                    }
                  
                }
            }else  if ("fieldses".equals(f.getName()) ){
                Struct[] paramses = new Struct[fieldses.size()];
                for (int i=0;i<fieldses.size();i++){
                    paramses[i]=fieldses.get(i).toStruct(conn);
                }
               ArrayDescriptor ad = new ArrayDescriptor("ARRAY_OF_FIELDS",conn);
                
                objs.add(new ARRAY(ad,conn,paramses));
            }
        }
        //使用java标准接口创建struct，对应一个plsql的object
       return     conn.createStruct("SERVICE_INFO",objs.toArray());
    }


}
```

调用：

```java
   stmt = am.getDBTransaction().createCallableStatement("begin import_service_from_excel.import(:1,:2);    end;", 1);
   stmt.registerOutParameter(2, Types.VARCHAR);
   Struct  serviceStruct =si.toStruct(stmt.getConnection());
   stmt.setObject(1, serviceStruct);
   stmt.execute();
   String msg = stmt.getString(2);
```



### plsql

##### object和array

> `object`中的字段需要和`java bean`中的声明字段顺序保持一致

```plsql
create or replace type service_fields as object
(
--字段名称
  field_name varchar2(200),
--类型
  field_type varchar2(100),
--长度
  field_length varchar2(20),
--位置
  field_position varchar2(100),
--层级
  field_level varchar2(10),
--描述
  field_desc varchar2(100),
--备注
  field_remarks varchar2(200)
)

create or replace type array_of_fields as array(1000000) of service_fields not null

create or replace type service_info as object
(
--服务号
  service_no varchar2(100),
--服务名
  service_name varchar2(100),

--服务版本
  serviceversion varchar2(50),
--服务提供方
  serviceprovider varchar2(50),
--联系人
  contact varchar2(50),
--一级分类
  aclass varchar2(50),
--二级分类
  bclass varchar2(50),
--三级分类
  cclass varchar2(50),
--服务描述
  servicedesc varchar2(150),
--业务场景
  businessscene varchar2(1000),
--备注
  remarks varchar2(200),
--触发系统报文
  triggeredmsg clob,
--接收系统报文
  receivemsg clob,
--定时/实时
  timingtype varchar2(20),
--全量/增量
  amounttype varchar2(20),
--同步/异步
  synctype varchar2(20),
--传输逻辑
  transferlogic varchar2(50),
--服务地址
  serviceaddress varchar2(1000),
--技术场景
  techscene varchar2(1000),
--参数字段
  fieldses array_of_fields

)
```

##### procedure

```plsql
procedure import(si  in service_info,
                   msg out varchar2) is
    l_cnt number;
  begin
   ...
    
  end import;
```


