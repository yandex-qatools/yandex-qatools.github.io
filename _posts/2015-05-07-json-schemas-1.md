---
layout: post
title:  "От json к json-схемам"
date:   2015-05-07 19:12:22 +0400
author:
    name: Kochkarev Aleksandr
    email: pleskav+blog@yandex.ru
    gravatar: 00000000000000000000000000000000
categories: [maven]
tags: [maven, json, jsonschema, beans]
comments: true
published: true
---

# Введение

В статье ["Генерируем бины по json описанию"](http://blog.qatools.ru/maven/json2pojo/) Кирилл познакомил нас с мавен-плагином *org.jsonschema2pojo:jsonschema2pojo-maven-plugin*.
Данный плагин позволяет генерировать бины как по самим json, так и по json-схемам.
Полученные java-объекты, как правило, используют для десериализации, но иногда и в качестве самостоятельного кода для служебных целей.

## Генерируем по json

Наиболее просто использовать данный плагин, когда для описания java-объектов достаточно json-а.

Возьмём, например, json c именем *pets.json*:
{% highlight json %}
{  
   "cat":{  
      "name":"Nancy",
      "food":"fish",
      "age":20,
      "kittens":[  
         "Marry",
         "Carry"
      ]
   },
   "dog":{  
      "name":"Sid",
      "food":"meat"
   }
}
{% endhighlight %}

Выставляем в json2pojo-плагине `<sourceType>` равный *json*; в `<targetPackage>` укажем, например, *com.example*; кладем наш файлик pets.json в `<sourceDirectory>`; компилируем проект	 и получаем в папке `<outputDirectory>` сгенерированный набор классов: 
  
  * `com.example.Cat` с полями *String name*, *String food*, *Integer age*, *List\<String\> kittens*,
  * `com.example.Dog` c полями *String name*, *String food*, 
  * `com.example.Example` с полями *Cat cat*, *Dog dog*.

Отметим, что имя файла *pets.json* повлияло на название класса `com.example.Pets`, название package *com.example*  берётся из `<targetPackage>`.

Посмотреть, что сгенерируется из любого json или json-схемы, можно на [сайте проекта jsonschema2pojo](http://www.jsonschema2pojo.org/)

## Зачем нам нужны json-схемы? 

К сожалению, если нужно сгенерировать java-объекты, которые отображают в себе структуру более сложных json-объектов, то с помощью способа представленного выше этого не всегда можно достичь.
Проще говоря, генерируя бины по json-у, объекты получаются не всегда такими, какими мы их хотим получить. 

Проиллюстрируем это на примере.
Дан json:
{% highlight json %}
{  
   "IPDATA":{  
      "date":"2014-12-25T12:54:03.433793",
      "iplist":{  
         "session:1411560095565656":[  
            "10.10.10.10",
            "10.10.10.9"
         ],
         "session:1411560095565659":[  
            "10.10.10.10"
         ]
      }
   }
}
{% endhighlight %}

Мы желаем получить объект `Map<String, java.util.List> iplist`, для которого `session:1411560095565656` и `session:1411560095565659` являлись бы ключами.
Но если сгенерировать бины по этому json-у, среди полей сгенерированных классов можно будет увидеть `List<String> session1411560095565656`
и `List<String> 1411560095565659`. 
Для преодоления данной проблемы как раз и используют json-схемы, которые позволяют описать json-объекты и соответствующие им java-объекты более гибко. 

##Учимся составлять json-схемы

Знакомство с правилами составления json-схем лучше начать с

  * [http://json-schema.org/](http://json-schema.org/) 
  * [http://json-schema.org/example1.html](http://json-schema.org/example1.html)
  * [http://json-schema.org/example2.html](http://json-schema.org/example2.html)

Примеры, представленные по этим ссылкам, помогут понять основные правила составления json-схем и 
без затруднений составить json-схему для описания знакомого нам *pets.json*:
{% highlight json %}
{  
   "type":"object",
   "properties":{  
      "cat":{  
         "type":"object",
         "properties":{  
            "name":{  
               "type":"string"
            },
            "food":{  
               "type":"string"
            },
            "age":{  
               "type":"integer"
            },
            "kittens":{  
               "type":"array",
               "minItems":1,
               "items":{  
                  "type":"string"
               },
               "uniqueItems":true
            }
         }
      },
      "dog":{  
         "type":"object",
         "properties":{  
            "name":{  
               "type":"string"
            },
            "food":{  
               "type":"string"
            }
         }
      }
   }
}
{% endhighlight %}

Конечно, иногда приходится составлять Json-схемы и посложнее, об этом во [второй части](/maven/json-schemas-2/). 

[gson]: https://code.google.com/p/google-gson/
[jaxb-artkoshelev]: http://artkoshelev.github.io/posts/jaxb-part-2/
[img-generated]: http://img-fotki.yandex.ru/get/9837/27441075.0/0_fda74_661f75fe_L.png
[github-json2pojo]: https://github.com/joelittlejohn/jsonschema2pojo
[jsonscheme]: http://json-schema.org/
[lambdaj]: http://code.google.com/p/lambdaj/
[json-path]: http://code.google.com/p/json-path/