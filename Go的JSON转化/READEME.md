# Go的JSON转化

## 结构体转JSON

```go
package main

import (
	"encoding/json" // 导入json包，提供了解码和编码JSON数据的函数
	"fmt"           // 导入fmt包，提供了格式化输入输出的函数
)

type Person struct {
	Name    string     `json:"name"`    // 使用结构体标签，定义在json数据中的字段名为"name"
	Age     int        `json:"age"`     // 定义在json数据中的字段名为"age"
	Address string     `json:"address"` // 定义在json数据中的字段名为"address"
	School  SchoolInfo `json:"school"`  // 定义在json数据中的字段名为"school"，字段类型是自定义的复合类型SchoolInfo
	// 注：字段的首字母大写是因为只有首字母大写的字段才会被外部包访问和识别，即使我们定义json后的小写字段名也是如此
}

type SchoolInfo struct {
	Name    string `json:"name"`    // 定义在json数据中的字段名为"name"
	Grade   int    `json:"grade"`   // 定义在json数据中的字段名为"grade"
	ClassNo int    `json:"classNo"` // 定义在json数据中的字段名为"classNo"
}

// 为Person类型定义一个方法，返回值是string和error类型
func (p Person) JSONString() (string, error) {
	// 使用json.Marshal来将复合类型的p转化成json，转化成后的数据类型是[]byte和error
	data, err := json.Marshal(&p)
	if err != nil { // 如果有错误（err不为nil）那么就返回空字符串和具体的错误信息
		return "", err
	}
	// 如果没有错误则返回将字节切片转化成的字符串和一个nil的error
	return string(data), nil
}

// 带格式化的JSON转化
func (p Person) JSONString2() (string, error) {
	// 使用json.MarshalIndent，并设置前缀为空，缩进为四个空格
	data, err := json.MarshalIndent(&p, "", "    ")
	if err != nil {
		return "", err
	}
	return string(data), nil
}

func main() {
	// 实例化一个Person类型的p
	p := Person{
		Name:    "大卫",
		Age:     10,
		Address: "北京市",
		School: SchoolInfo{
			// School字段是一个结构体类型，所以我们实例化一个SchoolInfo作为其值
			Name:    "上地实验小学",
			Grade:   4,
			ClassNo: 1,
		},
	}
	// 调用p的JSONString方法，得到json字符串和一个错误信息
	// jsonStr, err := p.JSONString()
	// 测试带格式化的转JSON
	jsonStr, err := p.JSONString2()
	if err != nil { // 判断错误信息是否为nil，如果不为nil，那么打印错误并返回
		fmt.Println("Error:", err)
		return
	}
	fmt.Println(jsonStr) // 打印json字符串
	// {"name":"大卫","age":10,"address":"北京市","school":{"name":"上地实验小学","grade":4,"classNo":1}}
	/*
		{
		    "name": "大卫",
		    "age": 10,
		    "address": "北京市",
		    "school": {
		        "name": "上地实验小学",
		        "grade": 4,
		        "classNo": 1
		    }
		}
	*/
}
```

* 导入`"encoding/json"`包

* 使用`json.Marshal`或`json.MarshalIndent`进行转化

* 将`[]byte`切片类型转为`string`类型



## 将JSON字符串转成结构体

```go
package main

import (
    "encoding/json"
    "fmt"
)

type Person struct {
    Name    string `json:"name"`
    Age     int    `json:"age"`
    Address string `json:"address"`
    School  SchoolInfo  `json:"school"`
}

type SchoolInfo struct {
    Name 	string 	`json:"name"`
    Grade 	int 	`json:"grade"`
    ClassNo int 	`json:"classNo"`
}

func main() {
    // 测试JSON字符串
    jsonStr := `{
        "name": "大卫",
        "age": 10,
        "address": "北京市",
        "school": {
            "name": "上地实验小学",
            "grade": 4,
            "classNo": 1
        }
    }`
    
    // 创建一个Person类型的变量p
    var p Person
    
    // 使用json.Unmarshal将jsonStr转化为p
    // 因为Unmarshal需要一个字节切片作为参数，所以我们需要将jsonStr转化为字节切片
    err := json.Unmarshal([]byte(jsonStr), &p)
    if err != nil {
        fmt.Println("Error:", err)
        return
    }
    // 打印p可以看到 jsonStr 被解析为Person结构体的结果
    fmt.Println(p)
    // {大卫 10 北京市 {上地实验小学 4 1}}
}
```

* 导入`"encoding/json"`包

* 使用`json.Unmarshal`将字符串转为结构体，需要将`string`转为`[]byte`类型


