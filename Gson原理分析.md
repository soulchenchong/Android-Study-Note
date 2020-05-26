Gson解析原理

- 调用**fromJson()**方法,最终会调用到下面这个方法

```
public <T> T fromJson(JsonReader reader, Type typeOfT) throws JsonIOException, JsonSyntaxException {
  boolean isEmpty = true;
  boolean oldLenient = reader.isLenient();
  reader.setLenient(true);
  try {
    reader.peek();
    isEmpty = false;
    TypeToken<T> typeToken = (TypeToken<T>) TypeToken.get(typeOfT);
    TypeAdapter<T> typeAdapter = getAdapter(typeToken);
    T object = typeAdapter.read(reader);
    return object;
  } catch (EOFException e) {
    /*
     * For compatibility with JSON 1.5 and earlier, we return null for empty
     * documents instead of throwing.
     */
    if (isEmpty) {
      return null;
    }
    throw new JsonSyntaxException(e);
  } catch (IllegalStateException e) {
    throw new JsonSyntaxException(e);
  } catch (IOException e) {
    // TODO(inder): Figure out whether it is indeed right to rethrow this as JsonSyntaxException
    throw new JsonSyntaxException(e);
  } catch (AssertionError e) {
    AssertionError error = new AssertionError("AssertionError (GSON " + GsonBuildConfig.VERSION + "): " + e.getMessage());
    error.initCause(e);
    throw error;
  } finally {
    reader.setLenient(oldLenient);
  }
}
```

关键代码如下:

- `reader.peek();`此方法会根据JsonReader读取的第一个字符来给JsonReader的成员属性peeked赋值;

- `TypeAdapter<T> typeAdapter = getAdapter(typeToken);` 根据需要序列化的对象类型得到一个typeAdapter,默认情况下,会获取到**ReflectiveTypeAdapterFactory.Adapter** ,这个adapter是ReflectiveTypeAdapterFactory的一个静态内部类

- `T object = typeAdapter.read(reader);` 这里实际就是调用**ReflectiveTypeAdapterFactory.Adapter** 的read()方法,接着看read()方法里做了什么

  ```
  @Override public T read(JsonReader in) throws IOException {
    if (in.peek() == JsonToken.NULL) {
      in.nextNull();
      return null;
    }
  
    T instance = constructor.construct();
  
    try {
      in.beginObject();
      while (in.hasNext()) {
        String name = in.nextName();
        BoundField field = boundFields.get(name);
        if (field == null || !field.deserialized) {
          in.skipValue();
        } else {
          field.read(in, instance);
        }
      }
    } catch (IllegalStateException e) {
      throw new JsonSyntaxException(e);
    } catch (IllegalAccessException e) {
      throw new AssertionError(e);
    }
    in.endObject();
    return instance;
  }
  ```

  + `T instance = constructor.construct(); ` 通过反射获取序列化对象类型的实例

  - 接着通过while(in.hasNext())不断去取json字符串里面的字符,这里有个boundFields类,下面看下这个类的含义

    ```
    private Map<String, BoundField> getBoundFields(Gson context, TypeToken<?> type, Class<?> raw) {
      Map<String, BoundField> result = new LinkedHashMap<String, BoundField>();
      if (raw.isInterface()) {
        return result;
      }
    
      Type declaredType = type.getType();
      while (raw != Object.class) {
        Field[] fields = raw.getDeclaredFields();
        for (Field field : fields) {
          boolean serialize = excludeField(field, true);
          boolean deserialize = excludeField(field, false);
          if (!serialize && !deserialize) {
            continue;
          }
          accessor.makeAccessible(field);
          Type fieldType = $Gson$Types.resolve(type.getType(), raw, field.getGenericType());
          List<String> fieldNames = getFieldNames(field);
          BoundField previous = null;
          for (int i = 0, size = fieldNames.size(); i < size; ++i) {
            String name = fieldNames.get(i);
            if (i != 0) serialize = false; // only serialize the default name
            BoundField boundField = createBoundField(context, field, name,
                TypeToken.get(fieldType), serialize, deserialize);
            BoundField replaced = result.put(name, boundField);
            if (previous == null) previous = replaced;
          }
          if (previous != null) {
            throw new IllegalArgumentException(declaredType
                + " declares multiple JSON fields named " + previous.name);
          }
        }
        type = TypeToken.get($Gson$Types.resolve(type.getType(), raw, raw.getGenericSuperclass()));
        raw = type.getRawType();
      }
      return result;
    }
    ```

    - BoundField是ReflectiveTypeAdapterFactory的抽象内部类,作用是包装解析的数据信息
    - 通过上面代码可以看到,主要是通过序列化对象的Class文件反射去拿到对应的成员变量,注解等信息,保存在BoundField中,然后以成员变量的名称为key,BoundField为value存在了一个Map中,需要注意下

    `createBoundField();` 这个方法

    ```
    private ReflectiveTypeAdapterFactory.BoundField createBoundField(
        final Gson context, final Field field, final String name,
        final TypeToken<?> fieldType, boolean serialize, boolean deserialize) {
      final boolean isPrimitive = Primitives.isPrimitive(fieldType.getRawType());
      // special casing primitives here saves ~5% on Android...
      JsonAdapter annotation = field.getAnnotation(JsonAdapter.class);
      TypeAdapter<?> mapped = null;
      if (annotation != null) {
        mapped = jsonAdapterFactory.getTypeAdapter(
            constructorConstructor, context, fieldType, annotation);
      }
      final boolean jsonAdapterPresent = mapped != null;
      if (mapped == null) mapped = context.getAdapter(fieldType);
    
      final TypeAdapter<?> typeAdapter = mapped;
      return new ReflectiveTypeAdapterFactory.BoundField(name, serialize, deserialize) {
        @SuppressWarnings({"unchecked", "rawtypes"}) // the type adapter and field type always agree
        @Override void write(JsonWriter writer, Object value)
            throws IOException, IllegalAccessException {
          Object fieldValue = field.get(value);
          TypeAdapter t = jsonAdapterPresent ? typeAdapter
              : new TypeAdapterRuntimeTypeWrapper(context, typeAdapter, fieldType.getType());
          t.write(writer, fieldValue);
        }
        @Override void read(JsonReader reader, Object value)
            throws IOException, IllegalAccessException {
          Object fieldValue = typeAdapter.read(reader);
          if (fieldValue != null || !isPrimitive) {
            field.set(value, fieldValue);
          }
        }
        @Override public boolean writeField(Object value) throws IOException, IllegalAccessException {
          if (!serialized) return false;
          Object fieldValue = field.get(value);
          return fieldValue != value; // avoid recursion for example for Throwable.cause
        }
      };
    }
    ```



注意`if (mapped == null) mapped = context.getAdapter(fieldType);  ` `final TypeAdapter<?> typeAdapter = mapped;` 默认情况下,会根据序列化对象里面的成员变量的类型去选择一个adapter,如果是String类型的成员变量,这里的typeAdapter实际上就是一个TypeAdapters.SING对象

```
public static final TypeAdapter<String> STRING = new TypeAdapter<String>() {
  @Override
  public String read(JsonReader in) throws IOException {
    JsonToken peek = in.peek();
    if (peek == JsonToken.NULL) {
      in.nextNull();
      return null;
    }
    /* coerce booleans to strings for backwards compatibility */
    if (peek == JsonToken.BOOLEAN) {
      return Boolean.toString(in.nextBoolean());
    }
    return in.nextString();
  }
  @Override
  public void write(JsonWriter out, String value) throws IOException {
    out.value(value);
  }
};
```

好了,我们现在回到typeAdapter.read();方法里

`      BoundField field = boundFields.get(name);
      if (field == null || !field.deserialized) {
        in.skipValue();
      } else {
        field.read(in, instance);
      }`

经过上面的分析我们知道boundFields是一个map对象,是以序列化对象的成员变量为key,去拿到对应的BoundField,接着调用BoundField.read()方法,会调到这

    @Override void read(JsonReader reader, Object value)
        throws IOException, IllegalAccessException {
      Object fieldValue = typeAdapter.read(reader);
      if (fieldValue != null || !isPrimitive) {
        field.set(value, fieldValue);
      }
    }
这里又会调用typeAdapter.read()方法,注意这里的typeAdapter我上面说了,会根据field的类型去选择对应的adpater,如果是String,就会调用STRING的read()方法,JsonToken peek = in.peek();
   ` if (peek == JsonToken.NULL) {
      in.nextNull();
      return null;
    }
    /* coerce booleans to strings for backwards compatibility */
    if (peek == JsonToken.BOOLEAN) {
      return Boolean.toString(in.nextBoolean());
    }
    return in.nextString();`

他会返回in.nextString(),就是对应json字符串的值,最后通过`field.set(value, fieldValue);` 反射设置field的值