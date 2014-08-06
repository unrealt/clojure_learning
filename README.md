# 对象建模与存储建模使用说明 #
----------
## <a name="ch1"></a>1. 快速入门 ##
对象建模与存储建模在使用上分为开发期和实施期两个阶段。

在开发期，主要关心模型的定义，需要进行以下操作：

1. 使用模型编辑工具开发[模型定义元数据](#user-content-ch3)。
2. 使用模型编辑工具开发[模型持久化定义元数据](#user-content-ch5)。（可选）
3. 在代码中使用[模型定义](#user-content-ch3.1)和[模型对象](#user-content-ch4)。

在初期阶段也可以使用工作副本接口，通过代码来定义模型和持久化定义，但不推荐。

在实施期，主要关心模型是怎样在实际数据库中进行存储，需要进行以下操作：

1. 使用实施工具，根据模型的定义和持久化定义，生成针对具体数据库的[存储模型定义元数据](#user-content-ch6)。
2. 根据实际情况，使用实施编辑工具手动修改存储模型定义。（可选）

## <a name="ch2"></a>2. 总体设计 ##
对象建模与存储建模分为模型定义、模型对象、持久化定义、存储模型等几个部分。其中使用者在接口层面上主要关心模型定义、模型对象、持久化定义这三个部分的接口，而存储模型则不对外提供接口形式的访问和修改方法。

在设计上，相关的具体类型和服务，全部以接口的形式提供，分为**只读接口**、**工作副本**（WorkingCopy）、**服务接口**这几个部分。

服务接口有只读的查询访问接口，和新建、修改、删除的管理接口两种，里面很多方法都有传Context和不传Context两种方式。如果不传Context，将通过ContextProvider来尝试从当前线程中获取context，如果失败将报错。

下文相关接口的示例代码，全部以有传context参数的接口为例子，更多的接口可以看具体的API。

## <a name="ch3"></a>3. 模型定义 ##
模型定义的接口分两部分，一部分是只读接口，主要是运行时使用。另外一部分是可以修改的工作副本接口，同样也是运行时使用的。

**初期阶段会同时开放提供只读接口和工作副本接口。在模型编辑工具开发好后，考虑只提供只读接口，屏蔽工作副本接口。**

### <a name="ch3.1"></a>3.1 模型定义只读接口 ###
只读接口是在 com.jiuqi.amino.model.define 包下面。下面是一些示例代码。

获取模型定义：

```java
ModelDefineServiceFacade mdsf = context.get(ModelDefineServiceFacade.class);
DomainModelDefine model = mdsf.getModel("model.full.name");
或
DomainModelDefine model = mdsf.getModel("package.name", "modelName");
```

列出模型上所有的属性定义，包括通过继承得到的来自父模型的所有属性：

```java
List<? extends AttributeDefine> attrs = model.listAllAttributes();
```

更详细的使用可以看具体的API。

### <a name="ch3.2"></a>3.2 模型定义工作副本接口 ###
所有对模型的修改都通过**模型定义工作副本**（DomainModelWorkingCopy）来进行。模型工作副本接口在 com.jiuqi.amino.model.define.workingcopy 包下面。下面是一些示例代码。

创建新的模型定义：

```java
ModelManageServiceFacade mmsf = context.get(ModelManageServiceFacade.class);
DomainModelWorkingCopy model = mmsf.createDomainModel("package.name", "modelName");
model.setPersistent(true);
AttributeDefineWorkingCopy attribute = mmsf.createAttribute("attributeName"); // 创建属性定义
StringTypeWorkingCopy strType = mmsf.createStringType(); // 创建字符串类型
strType.setSize(50); // 设置字符串大小不超过50
attribute.setType(strType); // 把属性设置为字符串类型
attribute.setUnique(true); // 设置属性值是唯一的
attribute.setNullable(false); // 设置属性值要求不能为空
model.addAttribute(attribute);
model.setPrimaryKey(mmsf.createPrimaryKey(Arrays.asList(attribute))); // 设置主键
mmsf.save(context, model);
```

获取已有模型的工作副本，有两种方法：

```java
DomainModelDefine model = ...; // 获取模型定义
DomainModelWorkingCopy copy = model.getWorkingCopy();
```

或者：

```java
ModelManageServiceFacade mmsf = context.get(ModelManageServiceFacade.class);
DomainModelWorkingCopy copy = mmsf.getModel("package.name", "modelName");
```

给已有模型定义添加一个新属性：

```java
DomainModelWorkingCopy copy = ...; // 获取模型定义工作副本
AttributeDefineWorkingCopy attribute = ...; // 创建一个新属性
copy.addAttribute(attribute);
ModelManageServiceFacade mmsf = context.get(ModelManageServiceFacade.class);
mmsf.save(context, copy);
```

修改模型定义上一个属性的类型及约束：

```java
AttributeDefineWorkingCopy attribute = copy.getAttribute("attributeName");
NumericTypeWorkingCopy numType = mmsf.createNumericType(); // 创建数值类型
numType.setPrecision(8, 0); // 设置精度为(8, 0)
attribute.setType(numType); // 修改属性的类型
attribute.setNullable(false); // 修改属性是否允许为空的设置
attribute.setDefaultValue(1000); // 修改属性默认值
ModelManageServiceFacade mmsf = context.get(ModelManageServiceFacade.class);
mmsf.save(context, copy);
```

更详细的使用可以看具体的API。

## <a name="ch4"></a>4. 模型对象 ##
模型对象的访问和使用有**动态对象**和**静态对象**两种方式。动态对象是一个固定的java类型，以get/set的方式访问对象的属性。静态对象是使用者自己定义的java类型，使用者可以详细指定对象里面属性的类型和访问方式。

初期阶段仅支持动态对象，后期再加上静态对象的支持。

### <a name="ch4.1"></a>4.1 动态对象 ###
动态对象的访问同样有只读接口和可修改接口（即工作副本）两种方式，所有接口都在 com.jiuqi.amino.model 包下面。

#### 4.1.1 动态对象只读接口 ####
动态对象的只读接口是 DomainObject 和 Attribute 两个，服务接口是 DomainObjectServiceFacade。下面是一些示例代码。

获取动态对象，以带context参数的访问方式为例：

```java
DomainObjectServiceFacade dosf = context.get(DomainObjectServiceFacade.class);
UUID id = ...; // 模型对象的ID
DomainObject dobj = dosf.getObject(context, "package.name", "modelName", id); // 获取模型下指定ID的对象实例
List<DomainObject> list = dosf.listObjects(context, "package.name", "modelName"); // 列出模型下所有的对象实例
int sum = list.parallelStream().mapToInt(dobj -> (Integer) dobj.getAttributeValue("intAttribute")).sum(); // Java 8，对所有对象实例的一个整形属性值求和
```

访问对象属性：

```java
DomainObject dobj = ...; // 获取动态对象
String str = (String) dobj.getAttributeValue("attributeName"); // 直接获取指定属性的值
Attribute attr = dobj.getAttribute("attributeName"); // 直接获取指定的属性
List<? extends Attribute> attrs = dobj.listAttributes(); // 列出模型上所有的属性
```

更详细的使用可以看具体的API。

#### 4.1.2 动态对象工作副本 ####
创建和修改动态对象，要通过工作副本来进行。主要的接口是 DomainObjectWorkingCopy 和 AttributeWorkingCopy，服务接口是 DomainObjectManageServiceFacade。下面是一些示例代码。

创建新的对象：

```java
DomainObjectManageServiceFacade domsf = context.get(DomainObjectManageServiceFacade.class);
DomainObjectWorkingCopy obj = domsf.createObject("package.name", "modelName"); // 创建指定模型的动态对象工作副本
obj.setAttributeValue("intAttribute", 100);
obj.setAttributeValue("stringAttribute", "value");

DomainObject refObj = ...; // 获取引用的对象
obj.setAttributeValue("refAttribute", refObj);

DomainObjectWorkingCopy subObj = domsf.createObject("package.name", "subModelName"); // 创建从模型的动态对象工作副本
... // 初始化从模型的属性
obj.setAttributeValue("subAttribute", Arrays.asList(subObj));

domsf.save(context, obj);
```

获取已有对象的工作副本，有两种方法：

```java
DomainObject dobj = ...; // 获取动态对象
DomainObjectWorkingCopy copy = dobj.getWorkingCopy();
```

或者：

```java
DomainObjectManageServiceFacade domsf = context.get(DomainObjectManageServiceFacade.class);
UUID id = ...; // 模型对象的ID
DomainObjectWorkingCopy copy = domsf.getObject(context, "package.name", "modelName", id);
```

修改已有对象的属性值并保存：

```java
DomainObjectWorkingCopy copy = ...; // 获取动态对象工作副本
copy.setAttributeValue("intAttribute", 100);
DomainObjectManageServiceFacade domsf = context.get(DomainObjectManageServiceFacade.class);
domsf.save(context, copy);
```

更详细的使用可以看具体的API。

### <a name="ch4.2"></a>4.2 静态对象 ###
静态对象是使用者自己定义的java类型，其定义和使用上要遵循一定的规范。

初期阶段对静态对象的支持仅限于提供未实现的接口，等静态对象支持完善之后，文档中本章节的内容会有调整。

#### 4.2.1 定义静态对象类型 ####
要声明一个静态对象类型，必须要加上 StaticDomainObject 的标签：

```java
@StaticDomainObject
public class MyStaticObject {
	// 详细定义
}
```

#### 4.2.2 定义静态对象类型的属性 ####
暂缺

#### 4.2.3 使用静态对象 ####
把动态对象转成静态对象：

```java
DomainObject dobj = ...; // 获取动态对象
MyStaticObject mso = dobj.convert(MyStaticObject.class);
```

直接以静态类型的方式从服务接口获取对象：

```java
DomainObjectServiceFacade dosf = context.get(DomainObjectServiceFacade.class);
UUID id = ...; // 模型对象的ID
MyStaticObject mso = dosf.getObject(MyStaticObject.class, "package.name", "modelName", id); // 以静态对象的方式获取模型下指定ID的对象实例
List<MyStaticObject> list = dosf.listObjects(MyStaticObject.class, "package.name", "modelName"); // 以静态对象的方式列出模型下所有的对象实例
```

保存静态对象：

```java
MyStaticObject mso = ...; // 创建或修改了一个静态对象
DomainObjectManageServiceFacade domsf = context.get(DomainObjectManageServiceFacade.class);
domsf.save(context, "package.name", "modelName", mso);
```

## <a name="ch5"></a>5. 模型持久化定义 ##
模型持久化定义的接口分为只读接口和工作副本接口两部分。

**初期阶段会同时开放提供只读接口和工作副本接口。在相关工具开发好后，考虑只提供只读接口，屏蔽工作副本接口。**

### <a name="ch5.1"></a>5.1 模型持久化定义只读接口 ###
只读接口是在 com.jiuqi.amino.model.define 包下面。下面是一些示例代码。

获取模型持久化定义：

```java
DomainModelDefine model = ...; // 获取具体的模型
ModelPersistentDefine mpd = model.getPersistentDefine(); // 获取模型对应的持久化定义，非持久化模型这里返回null
List<? extends IndexDefine> indexDefines = mpd.listIndexDefines(); // 列出所有的索引定义
List<? extends CombinedPersistentAttributeDefine> cpads = mpd.listCombinedPersistentAttributeDefines(); // 列出所有的组合持久化属性定义
```

更详细的使用可以看具体的API。

### <a name="ch5.2"></a>5.2 模型持久化定义工作副本接口 ###
所有对持久化模型的修改都通过**模型持久化定义工作副本**（ModelPersistentWorkingCopy）来进行。模型持久化定义工作副本接口在 com.jiuqi.amino.model.define.workingcopy 包下面。下面是一些示例代码。

创建新的模型持久化定义：

```java
ModelManageServiceFacade mmsf = context.get(ModelManageServiceFacade.class);
DomainModelWorkingCopy model = ...; // 获取模型工作副本
ModelPersistentWorkingCopy mpwc = mmsf.createPersistentDefine(); // 创建持久化定义工作副本
List<AttributeDefine> attributes = ...; // 准备索引的属性类表
IndexWorkingCopy index = mmsf.createIndex(attributes); // 创建索引定义工作副本
mpwc.addIndex(index);
model.setPersistentDefine(mpwc);
mmsf.save(context, model);
```

更详细的使用可以看具体的API。

## <a name="ch6"></a>6. 存储模型 ##
在DNA 4.0阶段，存储模型统一使用DNA逻辑表。存储模型的元数据是通过实施工具生成的，对外不提供读写访问接口。

存储模型的元数据包括：
* DNA逻辑表定义
* 存储模型与对象模型的映射关系

存储模型生成出来后，可以使用工具进行修改。允许修改的内容包括：
* 表字段名
* 分表逻辑

在运行时，存储模型元数据是由模型框架动态加载并发布应用到具体数据库的。如果数据库中该存储模型已经存在，则会比较是否有变化，并根据变化做调整。

