---
title: SpringBoot Json 浅析
date: 2021-07-24 10:34:53
categories:
  - Spring Boot
tags:
  - Spring Boot
  - json
---

# Spring Boot Json 浅析

```tex
在Spring中使用`@ResponseBody`注解可以将方法返回的对象序列化成JSON.

Spring Boot使用Jackson来完成JSON的序列化和反序列化操作.
```

## 简介

- @JsonIgnore 使用@JsonIgnore 注解的属性不会被序列化
- @JsonFormat(pattern = "yyyy-MM-dd") 指定序列化格式
- @JsonNaming 用于指定一个命名策略，作用于类或者属性上。Jackson 自带了多种命名策略，你可以实现自己的命名策略，比如输出的 key 由 Java 命名方式转为下面线命名方法 —— userName 转化为 user-name。
- @JsonView

### 自定义 ObjectMapper

```java
import java.text.SimpleDateFormat;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import com.fasterxml.jackson.databind.ObjectMapper;

@Configuration
public class JacksonConfig {
    @Bean
    public ObjectMapper getObjectMapper(){
        ObjectMapper mapper = new ObjectMapper();
        mapper.setDateFormat(new SimpleDateFormat("yyyy-MM-dd HH:mm:ss"));
        return mapper;
    }
}
```

### 定义对象

```java
public static class Car {
    private String name;

    @JsonIgnore
    private String band;

    @JsonFormat(pattern = "yyyy-MM-dd")
    private Date birth;

    private BigDecimal price;
}
```

### 对象序列化

```java
@Test
public void writeJson() throws JsonProcessingException {

  ObjectMapper mapper = new ObjectMapper();
  Car car = new Car();
  car.setName("A8L");
  car.setBand("Audi");
  car.setBirth(new Date());
  final String value = mapper.writeValueAsString(car);

  logger.info(value);
}
// 10:37:37.557 [Test worker] INFO zlin.site.framework.JackjsonTest - {"name":"A8L","birth":"2021-07-24"}
```

### 对象反序列化

```java
@Test
public void readJsonAsClassInstance() throws JsonProcessingException {
  ObjectMapper mapper = new ObjectMapper();

        String json = "{\"name\":\"A8L\",\"band\":\"Audi\",\"birth\":1627101288204,\"price\":900000}";
        final Car car = mapper.readValue(json, Car.class);
        logger.info(car.band);

        final JsonNode readTree = mapper.readTree(json);
        final String name = readTree.get("name").asText();
        logger.info("name: {}", name);

        // 凡序列化一个对象的属性为对象这种情况
        String jsonHasObjectProperties = "{\"name\":\"A8L\",\"birth\":\"2021-07-24\",\"price\":900000,\"color\":{\"name\":\"read\"}}";
        final JsonNode jsonNode = mapper.readTree(jsonHasObjectProperties);
        final String color = jsonNode.get("color").get("name").asText();
        logger.info("color={}",color);
}
```

### 集合对象反序列化

```java
// 在controller方法中会自动转为指定对象
@RequestMapping("updatecar")
@ResponseBody
public int updateUser(@RequestBody List<Car> list){
    return list.size();
}

// 这种方式不能将对象反序列化为指定类型对象
@Test
public void readJsonAsClassInstanceList() throws JsonProcessingException {
  ObjectMapper mapper = new ObjectMapper();

  String jsonArray = "[{\"name\":\"A8L\",\"band\":\"Audi\",\"birth\":1627101288204,\"price\":900000},{\"name\":\"A8L\",\"band\":\"Audi\",\"birth\":1627101288204,\"price\":900000}]";
  List<Car> list = mapper.readValue(jsonArray, List.class);
  for (Car car : list) {
    logger.info("name:{}", car.getBand());
  }
}
// java.util.LinkedHashMap cannot be cast to zlin.site.framework.JackjsonTest$Car
// java.lang.ClassCastException: java.util.LinkedHashMap cannot be cast to zlin.site.framework.JackjsonTest$Car

// 下面这种方式OK
@Test
public void readJsonAsClassInstanceList2() throws JsonProcessingException {
  ObjectMapper mapper = new ObjectMapper();

  String jsonArray = "[{\"name\":\"A8L\",\"band\":\"Audi\",\"birth\":1627101288204,\"price\":900000},{\"name\":\"A8L\",\"band\":\"Audi\",\"birth\":1627101288204,\"price\":900000}]";
  JavaType type = mapper.getTypeFactory().constructParametricType(List.class, Car.class);

  List<Car> list = mapper.readValue(jsonArray, type);
  for (Car car : list) {
    logger.info("name:{}", car.getName());
  }
}
```

### 自定义对象序列化

```java
// 自定义序列化
public static class PersonSerializer extends JsonSerializer<Person> {
  @Override
  public void serialize(Person person, JsonGenerator generator, SerializerProvider serializerProvider) throws IOException {
    generator.writeStartObject();
    generator.writeStringField("user-name", person.getUsername());
    generator.writeNumberField("age", person.getAge() + 20);
    generator.writeEndObject();
  }
}

// 使用指定的序列化方式
@JsonSerialize(using = PersonSerializer.class)
public static class Person implements Serializable {
  private String username;
  private String gender;
  private int age;
}

@Test
public void customizeSerializer() throws JsonProcessingException {
  Person person = new Person();
  person.setUsername("小汪");
  person.setGender("女");
  person.setAge(19);
  ObjectMapper mapper = new ObjectMapper();

  final String value = mapper.writeValueAsString(person);
  logger.info("Person={}", value);
}
// 13:17:56.356 [Test worker] INFO zlin.site.framework.JackjsonTest - Person={"user-name":"小汪","age":39}
```

### 自定义反序列化

```java
public static class PersonDeserializer extends JsonDeserializer<Person> {
  @Override
  public Person deserialize(JsonParser jsonParser, DeserializationContext deserializationContext) throws IOException, JsonProcessingException {
    JsonNode node = jsonParser.getCodec().readTree(jsonParser);
    String userName = node.get("user-name").asText();
    final int age = node.get("age").asInt();
    Person p = new Person();
    p.setUsername(userName);
    p.setGender("未知");
    p.setAge(age - 20);
    return p;
  }
}
// 在类上添加反序列化注解 @JsonDeserialize(using = PersonDeserializer.class)

@Test
public void customizeDeserializer() throws JsonProcessingException {
  ObjectMapper mapper = new ObjectMapper();
  String personJson = "{\"user-name\":\"小汪\",\"age\":39}";

  final Person person = mapper.readValue(personJson, Person.class);

  logger.info("person name = {}", person.getUsername());

}
// 13:29:32.478 [Test worker] INFO zlin.site.framework.JackjsonTest - person name = 小汪
```
