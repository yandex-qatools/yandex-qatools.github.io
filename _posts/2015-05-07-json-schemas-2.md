---
layout: post
title:  "Практикум по составлению json-схем"
date:   2015-05-07 19:12:22 +0400
author:
    name: Kochkarev Aleksandr
    email: pleskav+blog@yandex.ru
    gravatar: 00000000000000000000000000000000
categories: [maven]
tags: [maven, json,jsonschema, beans]
comments: true
published: true
---

В статье ["Генерируем бины по json описанию"](http://blog.qatools.ru/maven/json2pojo/) Кирилл познакомил нас с мавен-плагином *org.jsonschema2pojo:jsonschema2pojo-maven-plugin*.

В заметке ["От json к json-схемам"](/maven/json-schemas-1/) рассказано о том, как генерировать бины из json, json-схем и о том, в каких ситуациях json-схемы предпочтительней для этой генерации.

Далее хочу поделиться с вами свои опытом по составлению сложных json-схем.
    
## 1. Об опции плагина \<useLongIntegers\>

Тип объекта type в json-схеме может принимать следующие значения array, boolean,integer,number,null,object,string. ([JSON Schema: core definitions and terminology](http://json-schema.org/latest/json-schema-core.html#anchor8))

Попробуем составить json-схему для следующей json выдачи с использованием только стандартных type:
{% highlight json %}
{  
   "ts":1419503797,
   "ip":"194.186.25.31",
   "link":0,
   "ua":"Mozilla/5.0",
   "authid":{  
      "ip":"10.10.10.10",
      "host":"http://ho-st.host12324324.host"
   }
}
{% endhighlight %}


Такая json-схема будет иметь следующий вид:
{% highlight json linenos %}
{  
   "type":"object",
   "properties":{  
      "ts":{  
         "type":"integer"
      },
      "ip":{  
         "type":"string"
      },
      "link":{  
         "type":"integer"
      },
      "ua":{  
         "type":"string"
      },
      "authid":{  
         "type":"object",
         "properties":{  
            "ip":{  
               "type":"string"
            },
            "host":{  
               "type":"string"
            }
         }
      }
   }
}
{% endhighlight %}

Что нужно знать, о настройке плагина \<useLongIntegers\>?

Она влияет на переменные, чей `"type"`равен `"integer"`.

Вот в чём дело: если `useLongIntegers=false`, то переменные `link` и `ts`, тип которых мы указали как *integer*, в сгенерированных бинах будут приведены к *java.lang.Integer*;
если же выставим  `useLongIntegers=true`,  то переменные `link` и `ts` будут типа *java.lang.Long*.

Так что, когда вы работаете с большими числами в json-е, держите `useLongIntegers` включенным.

## 2.Подменяем стандартные type на  javaType

Как видно из предыдущего примера, настройка useLongIntegers позволяет сгенерировать все переменные с `"type":"integer"` либо  в java.lang.Long, либо в java.lang.Integer. 

А что же делать,  когда необходимо одну переменную получить типа Long, а вторую типа Integer? 
В такой ситуации удобно использовать javaType. 

javaType позволяет нам явно указать, что мы хотим получить в бинах в качестве типа переменной.
Json-схема с использованием javaType может выглядеть следующим образом:
{% highlight json linenos %}
{  
   "type":"object",
   "properties":{  
      "ts":{  
         "type":"object",
         "javaType":"java.lang.Long"
      },
      "ip":{  
         "type":"string"
      },
      "link":{  
         "type":"object",
         "javaType":"java.lang.Integer"
      },
      "ua":{  
         "type":"string"
      },
      "authid":{  
         "type":"object",
         "properties":{  
            "ip":{  
               "type":"string"
            },
            "host":{  
               "type":"string"
            }
         }
      }
   }
}
{% endhighlight %}

## 3.Увидел большую json-схему? Испугайся и разбей на несколько!
Честно признаюсь, когда я впервые увидел json-схему, мне она показалась невероятно громоздкой.
Дело в том, что json-схема всегда будет объемней того json-а, который она описывает.
И для крупных json-ов соответствующие им схемы нужно разбивать. 

Разбить json-схему на несколько файлов и сохранить при этом связность схемы можно с помощью `"$ref"`. ([JSON Schema: core definitions and terminology](http://json-schema.org/latest/json-schema-core.html#anchor30))
Так, например, представленную выше json-схему можно разнести по двум файлам следующим образом:

1) main.json:
{% highlight json linenos %}
{  
   "type":"object",
   "properties":{  
      "ts":{  
         "type":"object",
         "javaType":"java.lang.Long"
      },
      "ip":{  
         "type":"string"
      },
      "link":{  
         "type":"object",
         "javaType":"java.lang.Integer"
      },
      "ua":{  
         "type":"string"
      },
      "authid":{  
         "$ref":"authid.json"
      }
   }
}
{% endhighlight %}

2)authid.json
{% highlight json linenos %}
{  
   "id":"#authid",
   "type":"object",
   "properties":{  
      "ip":{  
         "type":"string"
      },
      "host":{  
         "type":"string"
      }
   }
}
{% endhighlight %}

## 4. Встраиваем с помощью javaType сгенерированные классы

В продолжении темы о разделении json-схем на части,отмечу очень важный бонус *javaType*:
в качестве *javaType* может быть указан объект, сгенерированный с помощью jsonschema2pojo по другой json-схеме и json-у.
Так у нас появляется возможность структурировать наши объекты ещё более гибко.

Вернемся к уже знакомому нам по первой части *pets.json*:
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
 
Для него мы можем сгенерировать объекты по совершенно иному принципу!

Легко видеть, что части *pets.json*, где перечисляются name,food,age,kittens, одинаковые по своему смыслу, это свойства животных кошки и собаки.
Поэтому неплохо было бы сгенерировать класс PetsProperties, описывающий эти свойства.
Для его генерации вынесем в отдельную json-схему *pets-properties.json*:  
{% highlight json linenos %}
{  
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
}
{% endhighlight %} 

Выставляем в плагине `<sourceType>jsonschema</sourceType>`; в `<targetPackage>` укажем com.example; кладем наш файлик pets-properties.json в `<sourceDirectory>`;
компилируем проект и получаем в папке `<outputDirectory>` сгенерированный набор класс com.example.PetsProperties с полями String name, String food, Integer age, List<String> kittens.

Этот класс нам пригодится, когда мы будем окончательно описывать **pets.json** с помощью вот такой json-cхемы (назовем его **new-pets.json**):  
{% highlight json linenos %}
{
  "type": "object",
  "properties": {
            "cat": {  "type": "object",
                      "javaType": "com.example.PetsProperty"
            },
            "dog": {  "type": "object",
                      "javaType": "com.example.PetsProperty"
            }
   }
 }
{% endhighlight %}

Аналогично описанному выше способу, получим объекты. Заметим, что сгенерированный класс com.example.NewPets будет содержать в себе два объекта *cat* и *dog*, только типа они будут одного *com.example.PetsProperty* (А не *com.example.Cat* и *com.example.Dog*, как мы получали раньше).
Чем это удобней?
Получаем меньше избыточности, меньше сгенерированного кода.


## 5. Обуздать коллекции вложенные друг в друга
Иногда приходится работать со сложными конструкциями в json,состоящими из 
объектов, в которых поля - это произвольные ключи, а их значение - произвольные объекты и т.д.

Такие сложные json нам требуется нередко представить в виде вложенных друг в друга коллекций: списков, состоящие из других списков, и map состоящие из других map и т.д 

Стандартными методами json-schema можно описывать коллекции большой степени вложенности с помощью `"type": "array"`:
{% highlight json linenos %}
(фрагмент)
{  
   "kittens":{  
      "type":"array",
      "uniqueItems":true,
      "minItems":1,
      "items":{  
         "type":"array",
         "minItems":1,
         "uniqueItems":true,
         "items":{  
            "type":"array",
            "minItems":1,
            "uniqueItems":true,
            "items":{  
               "type":"string"
            }
         }
      }
   }
}
{% endhighlight %}

Результатом такой json-схемы станет поле *Set\<Set\<Set\<String\>\>\> kittens = new LinkedHashSet\<Set\<Set\<String\>\>\>()* в сгенерированном классе.
Если убрать из схемы опции `"uniqueItems":true`, то получим *List\<List\<List\<String\>\>\> kittens= new ArrayList\<List\<List\<String\>\>\>()*.

Иногда требуется обыграть в json-е наличие структур типа *Map*. 

Вот, например, есть фрагмент такого json:
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

Для построения   соответствующей  json-схемы могу посоветовать использовать уже знакомый нам `javaType`:
{% highlight json linenos %}
{  
   "id":"#ipdata",
   "type":"object",
   "properties":{  
      "IPDATA":{  
         "type":"object",
         "properties":{  
            "date":{  
               "type":"string"
            },
            "iplist":{  
               "type":"object",
               "javaType":"java.util.Map<String, java.util.List>"
            }
         }
      }
   }
}
{% endhighlight %}

Начиная с версии 0.4.8 [jsonschema2pojo-плагина](https://github.com/joelittlejohn/jsonschema2pojo)  с помощью `javaType` можно задавать коллекции высокой степени вложенности, например:
`"javaType": "java.util.Map<String, java.util.Map<String, String>>"`,
`"javaType": "java.util.Map<String, java.util.Map<String, com.example.MyOwnBean>>"`.

## 6.Обуздать сложные конструкции с помощью кастомных десериализаторов
В предыдущем пункте я не просто так указал версию плагина, при котором `javaType` умеет работать с коллекциями высокой степени вложенности,
в версиях 0.4.7 и ниже `javaType` не позволял сделать такого. Мне в таком случае потребовалось написать собственный десериализатор. 

Покажу это на примере, есть такой json:
{% highlight json %}
{  
   "IPDATA":{  
      "data":{  
         "session1:1411560095":{  
            "0":{  
               "cnt":2,
               "last":1419589877000
            },
            "1":{  
               "cnt":1,
               "last":1419589877001
            }
         },
         "session2:1411560091":{  
            "0":{  
               "cnt":1,
               "last":1419589877003
            }
         }
      }
   }
}
{% endhighlight %}

Что мы здесь видим? 

Внутри `data` есть map, ключами которой являются элементы `session1:1411560095`,`session2:1411560091`.
Внутри `session1:1411560095` есть  map, ключами которой являются элементы `"0"`, `"1"`. Для `session2:1411560091` аналогично.

Очевидно, что для элементов вида `{"cnt": 1,"last": 1419589877000}` можно сразу составить отдельную json-схему session-detailed-information-bean.json и сгенерировать объект SessionDetailedInformationBean.

Схема *session-detailed-information-bean.json*:
{% highlight json linenos %}
{  
   "description":"schema for session items with detailed information",
   "type":"object",
   "properties":{  
      "cnt":{  
         "type":"object",
         "javaType":"java.lang.Integer"
      },
      "last":{  
         "type":"object",
         "javaType":"java.lang.Long"
      }
   }
}
{% endhighlight %}

Сгенерированный объект *SessionDetailedInformationBean*:
{% highlight java linenos %}
package com.example;

import javax.annotation.Generated;
import com.google.gson.annotations.Expose;
import org.apache.commons.lang3.builder.EqualsBuilder;
import org.apache.commons.lang3.builder.HashCodeBuilder;
import org.apache.commons.lang3.builder.ToStringBuilder;


/**
 * schema for session items with detailed information
 * 
 */
@Generated("org.jsonschema2pojo")
public class SessionDetailedInformationBean {

    @Expose
    private Integer cnt;
    @Expose
    private Long last;


    public Integer getCnt() {
        return cnt;
    }


    public void setCnt(Integer cnt) {
        this.cnt = cnt;
    }

    public SessionDetailedInformationBean withCnt(Integer cnt) {
        this.cnt = cnt;
        return this;
    }


    public Long getLast() {
        return last;
    }


    public void setLast(Long last) {
        this.last = last;
    }
}
{% endhighlight %}

Для описания всего json "IPDATA" вообще-то достаточно следующей json-схемы:

*ip-data.json*
{% highlight json linenos %}
{  
   "id":"#ipdata",
   "type":"object",
   "properties":{  
      "IPDATA":{  
         "type":"object",
         "properties":{  
            "data":{  
               "type":"object",
               "javaType":"java.util.Map<String, java.util.Map<String, com.example.SessionDetailedInformationBean>>"
            }
         }
      }
   }
}
{% endhighlight %}

Но такие сложные конструкции `java.util.Map<String, java.util.Map<String, com.example.SessionDetailedInformationBean>>` не поддерживались в плагине версии 0.4.7.

Чтобы выйти из сложившегося затруднения, мне пришлось предпринять следующие шаги:

### Шаг 1
Заменить `"javaType": "java.util.Map<String, java.util.Map<String, com.example.SessionDetailedInformationBean>>"` на допустимый
 `"javaType": "java.util.Map<String, com.example.SessionInformation>"`

### Шаг 2
Создать  класс *SessionInformation*, в котором, по сути, будет хранится *Map<String, com.example.SessionDetailedInformationBean>*. 

Например, так:
{% highlight java linenos %}
public class SessionInformation{
    private Map<String, SessionDetailedInformationBean> dataItem;

    public SessionInformation(Map<String, SessionDetailedInformationBean> dataItem) {
        this.dataItem = dataItem;
    }

    public Map<String, SessionDetailedInformationBean> getDataItem() {
        return dataItem;
    }

    public SessionInformation get(String sessionNumber) {
        return dataItem.get(sessionNumber);
    }
}
{% endhighlight %}


### Шаг 3
Создать кастомный десериализатор:

(Советую ознакомится со статьей [Gson или «Туда и Обратно»](http://habrahabr.ru/company/naumen/blog/228279/), она поясняет механизмы сериализации-десериализации).

{% highlight java linenos %}
import ch.lambdaj.function.convert.Converter;
import com.google.gson.JsonDeserializationContext;
import com.google.gson.JsonDeserializer;
import com.google.gson.JsonElement;
import com.google.gson.JsonParseException;
import com.example.SessionDetailedInformationBean;

import java.lang.reflect.Type;
import java.util.Map;

import static ch.lambdaj.Lambda.on;
import static ch.lambdaj.collection.LambdaCollections.with;

public class SessionInformationDeserializer implements JsonDeserializer<SessionInformation> {
    @Override
    public SessionInformation deserialize(JsonElement jsonElement, Type type,
                                  final JsonDeserializationContext context) throws JsonParseException {
        Map<String, SessionDetailedInformationBean> dataItemBeanMap =
                with(jsonElement.getAsJsonObject().entrySet()).map((on(Map.Entry.class).getKey().toString())).clone()
                        .convertValues(new Converter<Map.Entry<String, JsonElement>, SessionDetailedInformationBean>() {
                            @Override
                            public SessionDetailedInformationBean convert(Map.Entry<String, JsonElement> from) {
                                return context.deserialize(from.getValue(), SessionDetailedInformationBean.class);
                            }
                        });

        return new SessionInformation(dataItemBeanMap);
    }
}
{% endhighlight %}


### Шаг 4
Использовать десериализатор по назначению, так:
{% highlight java linenos %}
  Gson ipDataDeserializer = new GsonBuilder().registerTypeAdapter(SessionInformation.class, new SessionInformationDeserializer ()).create();
  Map<String, SessionInformation> data = ipDataDeserializer.fromJson(response.asString(), IpData.class)
                .getIPDATA().getData();               
{% endhighlight %}



[gson]: https://code.google.com/p/google-gson/
[jaxb-artkoshelev]: http://artkoshelev.github.io/posts/jaxb-part-2/
[img-generated]: http://img-fotki.yandex.ru/get/9837/27441075.0/0_fda74_661f75fe_L.png
[github-json2pojo]: https://github.com/joelittlejohn/jsonschema2pojo
[jsonscheme]: http://json-schema.org/
[lambdaj]: http://code.google.com/p/lambdaj/
[json-path]: http://code.google.com/p/json-path/