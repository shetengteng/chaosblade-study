Chaosblade 从 [0.1.0](https://github.com/chaosblade-io/chaosblade/releases/tag/0.1.0) 版本开始支持编写脚本实现复杂的 Java 实验场景，脚本支持 Java 和 Groovy 实现。本篇文章重点介绍此功能如何使用。

# 环境准备
* 下载最新版本 [chaosblade](https://github.com/chaosblade-io/chaosblade/releases/) 包，目前支持 Linux 和 Mac Darwin 平台。
* 准备代码编辑器，用于编写脚本。

# 使用方式
## 命令参数
执行 Java 实验场景前，必须执行 `prepare` 操作来安装 Java Agent，具体使用方式可通过 `./blade prepare jvm -h` 查看。动态脚本场景支持指定脚本文件或者脚本内容来调用，注意：脚本内容必须是 base64 编码后的内容。具体的使用帮助命令 `./blade create jvm script -h` :
```shell
Dynamically execute custom scripts

Usage:
  blade create jvm script

Flags:
      --classname string        The class name with package (required)
      --effect-count string     The count of chaos experiment in effect
      --effect-percent string   The percent of chaos experiment in effect
  -h, --help                    help for script
      --methodname string       The method name (required)
      --process string          Application process name
      --script-content string   The script contents
      --script-file string      The Script file full path
      --script-name string      The script name for label, unnecessary
      --script-type string      The script file type, java or groovy, default value is java
      --timeout string          set timeout for experiment

Global Flags:
  -d, --debug   Set client to DEBUG mode
```

参数介绍如下：
* `classname`: 必要参数，指定触发实验场景的类，注意，必须是实现类，不能是接口类。
* `methodname`: 必要参数，指定触发实验场景的方法，静态、私有、公有、父类方法都支持。
* `script-type`: 脚本类型，取值为 java 或 groovy，不加此参数，默认为 java。
* `script-file`: 脚本文件，文件绝对路径。
* `script-content`: 脚本内容，是 Base64 编码后的内容，相关工具类 [Base64Util](https://github.com/chaosblade-io/chaosblade-exec-jvm/blob/master/chaosblade-exec-plugin/chaosblade-exec-plugin-jvm/src/main/java/com/alibaba/chaosblade/exec/plugin/jvm/Base64Util.java)。注意，不能和 `script-file` 同时使用。
* `script-name`: 脚本名称，日志记录用，可不填写。
* `effect-count`: 影响的请求条数限制，Java 实验场景通用参数。
* `effect-percent`: 影响的请求百分比，0-100 的整数，可和 `effect-count` 共用，当达到 `effect-count` 限制时，`effect-percent` 不再起作用, Java 实验场景通用参数。
* `process`: 用来查找挂载 agent 的 java 进程，同 `prepare` 执行时, `--process` 的值一致。目的是区分单台机器多个 java 进程演练。 不填写，会默认查找第一个已挂载 agent 的进程。
* `timeout`: 实验场景运行时间，单位秒，到时间后会自动停止实验，通用参数。

## 执行 java 脚本实验场景
使用 `script-content` 指定演练脚本内容，不添加 `script-type` 参数，默认为 java 脚本，将调用 java 引擎解析器。
```shell
./blade c jvm script --classname com.example.controller.DubboController --methodname call --script-content aW1wb3J0IGphdmEudXRpbC5NYXA7CgppbXBvcnQgY29tLmV4YW1wbGUuY29udHJvbGxlci5DdXN0b21FeGNlcHRpb247CgovKioKICogQGF1dGhvciBDaGFuZ2p1biBYaWFvCiAqLwpwdWJsaWMgY2xhc3MgRXhjZXB0aW9uU2NyaXB0IHsKICAgIHB1YmxpYyBPYmplY3QgcnVuKE1hcDxTdHJpbmcsIE9iamVjdD4gcGFyYW1zKSB0aHJvd3MgQ3VzdG9tRXhjZXB0aW9uIHsKICAgICAgICBwYXJhbXMucHV0KCIxIiwgMTExTCk7CiAgICAgICAgLy9yZXR1cm4gIk1vY2sgVmFsdWUiOwogICAgICAgIC8vdGhyb3cgbmV3IEN1c3RvbUV4Y2VwdGlvbigiaGVsbG8iKTsKICAgICAgICByZXR1cm4gbnVsbDsKICAgIH0KfQo=  --script-name exception
```

使用 `script-file` 参数指定文件演练：
```shell
./blade c jvm script --classname com.example.controller.DubboController --methodname call --script-file /Users/Shared/IdeaProjects/Workspace_WebApp/dubbodemo/src/main/java/com/example/controller/ExceptionScript.java --script-name exception
```
执行成功，会返回 `{"code":200,"success":true,"result":"f614f5bdb279d905"}`，其中 `f614f5bdb279d905` 是实验 UID。

## 执行 groovy 脚本实验场景
参数同上，但必须添加 `--script-type groovy` 参数。如
```
./blade c jvm script --classname com.example.controller.DubboController --methodname call --script-file /Users/Shared/IdeaProjects/Workspace_WebApp/dubbodemo/src/main/java/com/example/controller/GroovyScript.groovy --script-name exception --script-type groovy 
```

# Java 脚本实现规范
##规范
* 必须创建一个类，对类名和包名没有要求，其中所依赖的类，必须是目标应用所具备的类。
* 必须添加 `public Object run(Map<String, Object> params)` 方法，其中 params 对象中包含目标方法参数，key 是参数索引下标，从 0 开始，比如目标方法是 `public String call(Object obj1, Object obj2){}`，则 `params.get("0")`则返回的是 `obj1` 对象，可以执行`params.put("0", <NEW OBJECT>)` 来修改目标方法参数（目标方法及 `--classname` 和 `--methodname` 所指定的类方法）。
* 上述方法返回的对象如果不为空，则会根据脚本中返回的对象来修改目标方法返回值，注意类型必须和目标方法返回值一致。如果上述方法返回 null，则不会修改目标方法返回值。

## 举例
对以下业务类做修改返回值实验场景：
```java
@RestController
@RequestMapping("/pet")
public class PetController {

    @GetMapping("/list")
    public Result<List<PetVO>> getPets() {
        Map<Long, Discount> petDiscount = discountManager
            .getPetDiscounts()
            .stream()
            .filter(discount -> discount.getExpired() == 0)
            .collect(Collectors.toMap(
                Discount::getPetId,
                Function.identity()
            ));

        List<PetVO> pets = petManager
            .getPets()
            .stream()
            .map(pet -> {
                PetVO petVO = PetVO.from(pet);
                Discount discount = petDiscount.get(pet.getId());

                if (null != discount && null != discount.getDiscountPrice() && discount.getDiscountPrice() > 0L) {
                    petVO.setDiscountPrice(discount.getDiscountPrice());
                }

                return petVO;
            })
            .collect(Collectors.toList());

       return Result.success(pets);
    }
```
则编写 Java 脚本，实现对 `getPets` 方法做返回值修改：
```java
package com.alibaba.csp.monkeyking.controller;

import java.util.ArrayList;
import java.util.List;
import java.util.Map;

import com.alibaba.csp.monkeyking.demo.model.Pet;
import com.alibaba.csp.monkeyking.model.PetVO;
import com.alibaba.csp.monkeyking.model.Result;

public class ChaosController {

    public Object run(Map<String, Object> params) {
        ArrayList<PetVO> petVOS = new ArrayList<>();
        for (int i = 0; i < 3; i++) {
            Pet pet = new Pet();
            pet.setName("test_" + i);
            PetVO petVO = PetVO.from(pet);
            petVOS.add(petVO);
        }
        Result<List<PetVO>> results = Result.success(petVOS);
        return results;
    }
}
```

保存文件后，通过上面 `使用方式` 部分的命令来调用，也可以将其进行 Base64 编码，通过指定 `script-content` 参数来指定编码后的内容。

## 效果
### 未执行实验之前页面：
![demo 页面](https://user-images.githubusercontent.com/3992234/59833022-c494fb00-9377-11e9-8d14-e3ad31b0acea.png)

### 执行实验之后页面：
![demo 页面-演练后](https://user-images.githubusercontent.com/3992234/59833094-ee4e2200-9377-11e9-9b24-41f5f30053e0.png)


# 问题排查
Java 实验场景的日志在 进程用户下 logs/chaosblade/chaosblade.log 中。执行脚本成功，但不生效，原因可能是脚本编译错误（因为脚本编译方法调用时触发，所以下发脚本，不会进行编译），可查看此日志进行排查。

