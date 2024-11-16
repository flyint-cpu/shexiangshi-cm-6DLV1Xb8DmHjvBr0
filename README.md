
JSON.parse 是我们在前端开发中经常会用到API，如果我们要自己实现一个JSON.parse，我们应该怎么实现呢？今天我们就试着手写一个JSON Parser，了解下其内部实现原理。


## JSON语法


JSON 是一种语法，用来序列化对象、数组、数值、字符串、布尔值和 null 。语法规则如下：


* 数据使用名/值对表示。
* 使用大括号({})保存对象，每个名称后面跟着一个 ':'(冒号)，名/值对使用 ,(逗号)分割。


![file](https://img2024.cnblogs.com/other/2332333/202411/2332333-20241115104008063-1158405538.png)


* 使用方括号(\[])保存数组，数组值使用 ,(逗号)分割。


![file](https://img2024.cnblogs.com/other/2332333/202411/2332333-20241115104008364-281648810.png)


* JSON值可以是：数字(整数或浮点数)/字符串(在双引号中)/逻辑值(true 或 false)/数组(在方括号中)/对象(在花括号中)/null


![file](https://img2024.cnblogs.com/other/2332333/202411/2332333-20241115104008812-553674328.png)


## 实现Parser


Parser 一般会经过下面几个过程，分为**词法分析 、语法分析、转换、代码生成**过程。


### 词法分析


![file](https://img2024.cnblogs.com/other/2332333/202411/2332333-20241115104009244-1011200272.png)


通过对 JSON 语法的了解，我们可以看到 JSON 中会有一下类型及其特征如下表：




| 类型 | 基本特征 |
| --- | --- |
| Object | "{" ":" "," "}" |
| Array | "\[" "," "]" |
| String | '"' |
| Number | "0" "1" "2" "3" "4" "5" "6" "7" "8" "9" |
| Boolean | "true" "false" |
| Null | "null" |


所以根据这些特征，对 JSON 字符串进行遍历操作并与上述特征进行对比可以得到相应的 token。词法分析实现代码如下：



```
// 词法分析
const TokenTypes = {
  OPEN_OBJECT: '{',
  CLOSE_OBJECT: '}',
  OPEN_ARRAY: '[',
  CLOSE_ARRAY: ']',
  STRING: 'string',
  NUMBER: 'number',
  TRUE: 'true',
  FALSE: 'false',
  NULL: 'null',
  COLON: ':',
  COMMA: ',',
}

class Lexer {
  constructor(json) {
    this._json = json
    this._index = 0
    this._tokenList = []
  }

  createToken(type, value) {
    return { type, value: value || type }
  }

  getToken() {
    while (this._index < this._json.length) {
      const token = this.bigbang()
      this._tokenList.push(token)
    }
    return this._tokenList
  }

  bigbang() {
    const key = this._json[this._index]
    switch (key) {
      case ' ':
        this._index++
        return this.bigbang()
      case '{':
        this._index++
        return this.createToken(TokenTypes.OPEN_OBJECT)
      case '}':
        this._index++
        return this.createToken(TokenTypes.CLOSE_OBJECT)
      case '[':
        this._index++
        return this.createToken(TokenTypes.OPEN_ARRAY)
      case ']':
        this._index++
        return this.createToken(TokenTypes.CLOSE_ARRAY)
      case ':':
        this._index++
        return this.createToken(TokenTypes.COLON)
      case ',':
        this._index++
        return this.createToken(TokenTypes.COMMA)
      case '"':
        return this.parseString()
    }
    // number
    if (this.isNumber(key)) {
      return this.parseNumber()
    }
    // true false null
    const result = this.parseKeyword(key)
    if (result.isKeyword) {
      return this.createToken(TokenTypes[result.keyword])
    }
  }

  isNumber(key) {
    return key >= '0' && key <= '9'
  }

  parseString() {
    this._index++
    let key = ''
    while (this._index < this._json.length && this._json[this._index] !== '"') {
      key += this._json[this._index]
      this._index++
    }
    this._index++
    return this.createToken(TokenTypes.STRING, key)
  }

  parseNumber() {
    let key = ''
    while (this._index < this._json.length && '0' <= this._json[this._index] && this._json[this._index] <= '9') {
      key += this._json[this._index]
      this._index++
    }
    return this.createToken(TokenTypes.NUMBER, Number(key))
  }

  parseKeyword(key) {
    let isKeyword = false
    let keyword = ''
    switch (key) {
      case 't':
        isKeyword = this._json.slice(this._index, this._index + 4) === 'true'
        keyword = 'TRUE'
        break
      case 'f':
        isKeyword = this._json.slice(this._index, this._index + 5) === 'false'
        keyword = 'FALSE'
        break
      case 'n':
        isKeyword = this._json.slice(this._index, this._index + 4) === 'null'
        keyword = 'NULL'
        break
    }
    this._index += keyword.length
    return {
      isKeyword,
      keyword,
    }
  }
}

```

### 语法分析


![file](https://img2024.cnblogs.com/other/2332333/202411/2332333-20241115104009870-1611994307.png)


语法分析是遍历每个 Token，寻找语法信息，并且构建一个叫做 AST(抽象语法树)的对象。在正式进行语法分析前，我们针对 JSON 的语法特征创建不同的类来记录 AST 上每个节点的信息。



```
class NumericLiteral {
  constructor(type, value) {
    this.type = type
    this.value = value
  }
}

class StringLiteral {
  constructor(type, value) {
    this.type = type
    this.value = value
  }
}

class BooleanLiteral {
  constructor(type, value) {
    this.type = type
    this.value = value
  }
}

class NullLiteral {
  constructor(type, value) {
    this.type = type
    this.value = value
  }
}

class ArrayExpression {
  constructor(type, elements) {
    this.type = type
    this.elements = elements || []
  }
}

class ObjectExpression {
  constructor(type, properties) {
    this.type = type
    this.properties = [] || properties
  }
}

class ObjectProperty {
  constructor(type, key, value) {
    this.type = type
    this.key = key
    this.value = value
  }
}

```

接下来正式进行语法分析，对 Token 进行遍历并对其类型进行检查，创建节点信息，构建一个 AST(抽象语法树)的对象。代码如下：



```
// 语法分析
class Parser {
  constructor(tokens) {
    this._tokens = tokens
    this._index = 0
    this.node = null
  }

  jump() {
    this._index++
  }

  getValue() {
    const value = this._tokens[this._index].value
    this._index++
    return value
  }

  parse() {
    const type = this._tokens[this._index].type
    const value = this.getValue()
    switch (type) {
      case TokenTypes.OPEN_ARRAY:
        const array = this.parseArray()
        this.jump()
        return array
      case TokenTypes.OPEN_OBJECT:
        const object = this.parseObject()
        this.jump()
        return object
      case TokenTypes.STRING:
        return new StringLiteral('StringLiteral', value)
      case TokenTypes.NUMBER:
        return new NumericLiteral('NumericLiteral', Number(value))
      case TokenTypes.TRUE:
        return new BooleanLiteral('BooleanLiteral', true)
      case TokenTypes.FALSE:
        return new BooleanLiteral('BooleanLiteral', false)
      case TokenTypes.NULL:
        return new NullLiteral('NullLiteral', null)
    }
  }

  parseArray() {
    const _array = new ArrayExpression('ArrayExpression')
    while(true) {
      const value = this.parse()
      _array.elements.push(value)
      if (this._tokens[this._index].type !== TokenTypes.COMMA) break
      this.jump() // 跳过 ,
    }
    return _array
  }

  parseObject() {
    const _object = new ObjectExpression('ObjectExpression')
    _object.properties = []
    while(true) {
      const key = this.parse()
      this.jump() // 跳过 : 
      const value = this.parse()
      const property = new ObjectProperty('ObjectProperty', key, value)
      _object.properties.push(property)
      if (this._tokens[this._index].type !== TokenTypes.COMMA) break
      this.jump() // 跳过 ,
    }
    return _object
  }
}

```

### 转换


经过语法分析后得到了 AST，转换阶段可以对树节点进行增删改等操作，转换为新的 AST 树。


### 代码生成


生成代码阶段，是对转换后的 AST 进行遍历，根据每个节点的语法信息转换成最终的代码。



```
// 代码生成
class Generate {
  constructor(tree) {
    this.tree = tree
  }

  getResult() {
    let result = this.getData(this.tree)
    return result
  }

  getData(data) {
    if (data.type === 'ArrayExpression') {
      let result = []
      data.elements.map(item => {
        let element = this.getData(item)
        result.push(element)
      })
      return result
    }
    if (data.type === 'ObjectExpression') {
      let result = {}
      data.properties.map(item => {
        let key = this.getData(item.key)
        let value = this.getData(item.value)
        result[key] = value
      })
      return result
    }
    if (data.type === 'ObjectProperty') {
      return this.getData(data)
    }
    if (data.type === 'NumericLiteral') {
      return data.value
    }
    if (data.type === 'StringLiteral') {
      return data.value
    }
    if (data.type === 'BooleanLiteral') {
      return data.value
    }
    if (data.type === 'NullLiteral') {
      return data.value
    }
  }
}

```

### 使用



```
function JsonParse(b) {
  const lexer = new Lexer(b)
  const tokens = lexer.getToken() // 获取Token
  const parser = new Parser(tokens)
  const tree = parser.parse() // 生成语法树
  const generate = new Generate(tree)
  const result = generate.getResult() // 生成代码
  return result
}

```

## 总结


至此我们就实现了一个简单的 JSON Parse 解析器，通过对 JSON Parse 实现的探究，我们可以总结出此类解析器的实现步骤，首先对目标值的语法进行了解，提取其特征，然后通过词法分析，与目标特征进行比对得到 token，然后对 token 进行语法分析生成 AST(抽象语法树)，再对 AST 进行增删改等操作，生成新的 AST，最终对 AST 进行遍历就会生成我们需要的目标值。


## 参考


* [https://www.json.org/json\-en.html](https://github.com)
* [https://lihautan.com/json\-parser\-with\-javascript/](https://github.com)
* [https://developer.mozilla.org/zh\-CN/docs/Web/JavaScript/Reference/Global\_Objects/JSON](https://github.com)


## 最后


欢迎关注【袋鼠云数栈UED团队】\~
袋鼠云数栈 UED 团队持续为广大开发者分享技术成果，相继参与开源了欢迎 star


* **[大数据分布式任务调度系统——Taier](https://github.com)**
* **[轻量级的 Web IDE UI 框架——Molecule](https://github.com):[FlowerCloud机场](https://hanlianfangzhi.com)**
* **[针对大数据领域的 SQL Parser 项目——dt\-sql\-parser](https://github.com)**
* **[袋鼠云数栈前端团队代码评审工程实践文档——code\-review\-practices](https://github.com)**
* **[一个速度更快、配置更灵活、使用更简单的模块打包器——ko](https://github.com)**
* **[一个针对 antd 的组件测试工具库——ant\-design\-testing](https://github.com)**


