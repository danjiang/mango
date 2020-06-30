---
title: Swift 进阶 - 编码和解码
author: 但江
avatar: danjiang
location: 成都
category: programming
tag: swift
---

Swift 中有如数字、字符串、数组、字典和集合等内置的数据结构，当需要和其他数据结构如 JSON 和 PropertyList，互相进行转换时，就会涉及到编码和解码。

![Advance Swift](/images/advance-swift.png)

## 编码和解码的协议 

{% highlight swift %}
protocol Encodable {
    // 把值编码到 encoder 中
    func encode(to encoder: Encoder) throws
}

protocol Decodable {
    // 从 decoder 中把值解码出来
    init(from decoder: Decoder) throws
}

typealias Codable = Decodable & Encodable
{% endhighlight %}

## JSON 编码和解码

### 满足 Codable 协议

如下代码中，GroceryProduct 只包含基础数据类型，所以编译器可以自动生成满足 Codable 协议的代码：

{% highlight swift %}
struct GroceryProduct: Codable {
    var name: String
    var points: Int
    var description: String?
}
{% endhighlight %}

### JSONEncoder

{% highlight swift %}
let pear = GroceryProduct(name: "Pear", points: 250, description: "A ripe pear.")

let encoder = JSONEncoder()
encoder.outputFormatting = .prettyPrinted

let data = try encoder.encode(pear)
print(String(data: data, encoding: .utf8)!)

/* Prints:
 {
   "name" : "Pear",
   "points" : 250,
   "description" : "A ripe pear."
 }
*/
{% endhighlight %}

### JSONDecoder

{% highlight swift %}
let json = """
{
    "name": "Durian",
    "points": 600,
    "description": "A fruit with a distinctive scent."
}
""".data(using: .utf8)!

let decoder = JSONDecoder()
let product = try decoder.decode(GroceryProduct.self, from: json)

print(product.name) // Prints "Durian"
{% endhighlight %}

## 编码过程详解

{% highlight swift %}
struct Coordinate: Codable {
    var latitude: Double
    var longitude: Double
}

struct Placemark: Codable {
    var name: String
    var coordinate: Coordinate
}

let places = [
    Placemark(name: "Berlin", coordinate: Coordinate(latitude: 52, longitude: 13)),
    Placemark(name: "Cape Town", coordinate: Coordinate(latitude: -34, longitude: 18))
]

do {
    let encoder = JSONEncoder()
    let jsonData = try encoder.encode(places) // 👈 pay attention
} catch {
    print(error.localizedDescription)
}
{% endhighlight %}

如上代码中，encode 这一步会转换成如下代码，places 是 Array，会调用 Array 对于 encode(to:) 的实现：

{% highlight swift %}
places.encode(to: self)
{% endhighlight %}

Array 对于 Encodable 协议的实现大概如下，先得到 UnkeyedEncodingContainer 容器，然后遍历 Array 中的元素，将其装到容器中，UnkeyedEncodingContainer 容器中 encode 的实现会判断 Array 中的元素，如果是基础数据类型，会用 SingleValueEncodingContainer 容器将数据装起来，如果不是基础数据类型时，会继续调用其 encode(to:) 方法，就这样层层转换下去，直到基础数据类型：

{% highlight swift %}
extension Array: Encodable where Element: Encodable {
    public func encode(to encoder: Encoder) throws {
        var container = encoder.unkeyedContainer() for element in self {
            try container.encode(element)
        }
    }
}
{% endhighlight %}

上面简单说了 SingleValueEncodingContainer 容器，Encoder 协议中共有如下 3 种容器：

* KeyedEncodingContainer 类似于字典，键值对容器，键值是强类型。
* UnkeyedEncodingContainer 类似于数组，连续值容器，没有键值。
* SingleValueEncodingContainer 基础数据类型容器。

{% highlight swift %}
protocol Encoder {
    container<Key>(keyedBy: Key.Type) -> KeyedEncodingContainer<Key>
    unkeyedContainer() -> UnkeyedEncodingContainer
    singleValueContainer() -> SingleValueEncodingContainer
}
{% endhighlight %}

可以得出，整个编码的过程是将 Swift 数据结构层层转换为容器，容器转换为其他数据结构如 JSON 和 PropertyList。

## 满足 Codable 协议的代码生成

编译器可以自动生成满足 Codable 协议的代码：

### 生成 Coding Keys

{% highlight swift %}
struct Placemark {
    private enum CodingKeys: CodingKey {
        case name
        case coordinate
    }
}
{% endhighlight %}

### 生成 encode(to:)

可以看到如下代码中，使用的是 KeyedEncodingContainer 容器：

{% highlight swift %}
struct Placemark: Codable {
    func encode(to encoder: Encoder) throws {
        var container = encoder.container(keyedBy: CodingKeys.self)
        try container.encode(name, forKey: .name)
        try container.encode(coordinate, forKey: .coordinate)
    }
}
{% endhighlight %}

### 生成 init(from:)

这里是关于解码的代码，从容器中获取 Swift 数据结构：

{% highlight swift %}
struct Placemark: Codable {
    init(from decoder: Decoder) throws {
        let container = try decoder.container(keyedBy: CodingKeys.self)
        name = try container.decode(String.self, forKey: .name)
        coordinate = try container.decode(Coordinate.self, forKey: .coordinate)
    }
}
{% endhighlight %}

同编码一样，解码相关的 Decoder 协议也有如下 3 个容器：

{% highlight swift %}
protocol Decoder {
    container<Key>(keyedBy type: Key.Type) throws -> KeyedDecodingContainer<Key>
    unkeyedContainer() throws -> UnkeyedDecodingContainer
    singleValueContainer() throws -> SingleValueDecodingContainer
}
{% endhighlight %}

## 自己实现 Codable 协议

根据实际的需求，有时候需要自己实现或部分实现 Codable 协议。

### 自定义 Coding Keys

enum CodingKeys 中 case 就是需要编解码，不再其中就不需要编解码，还可以修改转换时对应的键值，可以看到如果只修改其中几项，全部都要写一遍，非常的冗长。

{% highlight swift %}
class ItemModel: Codable {
    var id: String?
    var fullName: String?
    var shortName: String?
    var description: String?
    var displayDescription: String?
    var shareDescription: String?
    var activityItemNo: String?
    var activityUUID: String?
    var listPrice: String?
    var skus: [SkuModel]?
    var inStockSkus: [SkuModel] = []
    var images: [String]?
    var mainImageIndex: String?
    var itemMedias: [String]?
    var unfold = false
    var increasePrice: Decimal?
    var increaseType: ShareIncreaseType?
    var isShared = false
    var reloading = false
    var hasCustomService = false
    var memberLevel: MemberLevelType?
    var nextMemberLevel: MemberLevelType?
    var medias: [String]?
    
    enum CodingKeys: String, CodingKey {
        case id
        case fullName
        case shortName
        case description
        case displayDescription
        case shareDescription
        case activityItemNo
        case activityUUID = "activityUuid"
        case listPrice
        case skus
        case images
        case itemMedias
        case mainImageIndex
        case medias
    }
}
{% endhighlight %}

### 自定义 init(from:)

当要处理如下不标准 JSON 数据时，coordinate 不存在时返回的是空 JSON 对象：

{% highlight swift %}
let invalidJSONInput = """
[
    {
        "name" : "Berlin",
        "coordinate": {}
    }
]
"""
{% endhighlight %}

如下代码捕获了 coordinate 解码不成功的异常，将 coordinate 赋值为 nil：

{% highlight swift %}
struct Placemark: Codable {
    var name: String
    var coordinate: Coordinate
    
    init(from decoder: Decoder) throws {
        let container = try decoder.container(keyedBy: CodingKeys.self)
        self.name = try container.decode(String.self, forKey: .name)
        do {
            self.coordinate = try container.decodeIfPresent(Coordinate.self, forKey: .coordinate)
        } catch DecodingError.keyNotFound {
            self.coordinate = nil
        }
    }
}
{% endhighlight %}

如果你对其他数据结构如 JSON 和 PropertyList 的来源方有足够的掌控，数据交换就可以按照数据格式语义来，不会有什么问题，如果没有足够的掌控，也许就需要做很多上面的自定义工作，就比较麻烦。
