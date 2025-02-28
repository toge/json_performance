# json_performance
Performance profiling of JSON libraries

Latest results (March 1, 2023):

| Library                                                      | Roundtrip Time (s) | Write (MB/s) | Read (MB/s) |
| ------------------------------------------------------------ | ------------------ | ------------ | ----------- |
| [**Glaze**](https://github.com/stephenberry/glaze)           | **1.30**           | **907**      | **941**     |
| [**simdjson (on demand)**](https://github.com/simdjson/simdjson) | **N/A**            | **N/A**      | **1257**    |
| [**yyjson**](https://github.com/ibireme/yyjson)              | **1.73**           | **633**      | **1021**    |
| [**daw_json_link**](https://github.com/beached/daw_json_link) | **2.79**           | **382**      | **487**     |
| [**RapidJSON**](https://github.com/Tencent/rapidjson)        | **3.21**           | **311**      | **630**     |
| [**json_struct**](https://github.com/jorgen/json_struct)     | **4.29**           | **236**      | **329**     |
| [**nlohmann**](https://github.com/nlohmann/json)             | **17.08**          | **89**       | **72**      |

>  1,000,000 iterations on a single core (MacBook Pro M1)

Test object (minified for test):

```json
{
   "fixed_object": {
      "int_array": [0, 1, 2, 3, 4, 5, 6],
      "float_array": [0.1, 0.2, 0.3, 0.4, 0.5, 0.6],
      "double_array": [3288398.238, 233e22, 289e-1, 0.928759872, 0.22222848, 0.1, 0.2, 0.3, 0.4]
   },
   "fixed_name_object": {
      "name0": "James",
      "name1": "Abraham",
      "name2": "Susan",
      "name3": "Frank",
      "name4": "Alicia"
   },
   "another_object": {
      "string": "here is some text",
      "another_string": "Hello World",
      "boolean": false,
      "nested_object": {
         "v3s": [[0.12345, 0.23456, 0.001345],
                  [0.3894675, 97.39827, 297.92387],
                  [18.18, 87.289, 2988.298]],
         "id": "298728949872"
      }
   },
   "string_array": ["Cat", "Dog", "Elephant", "Tiger"],
   "string": "Hello world",
   "number": 3.14,
   "boolean": true,
   "another_bool": false
}
```

## ABC Test (Out of Sequence Performance)

In this test the JSON document has keys from "a" to "z", where each key refers to an array of integers from [0, 999]. The document orders the keys from "z" to "a", in reverse order to the expected "a" to "z" layout.

This test demonstrates problems with `simdjson` and iterative parsers that cannot hash for memory locations. If keys are not in the expected sequence performance seriously suffers, and the problem increases as the size of the document increases.

Hash based solutions avoid this problem and do not suffer performance loss as the JSON document grows in size.

| Library                                                      | Roundtrip Time (s) | Write (MB/s) | Read (MB/s) |
| ------------------------------------------------------------ | ------------------ | ------------ | ----------- |
| [**Glaze**](https://github.com/stephenberry/glaze)           | **2.44**           | **1334**     | **564**     |
| [**simdjson (on demand)**](https://github.com/simdjson/simdjson) | **N/A**            | **N/A**      | **114**     |

# Registration Comparison

### Glaze

```c++
template <>
struct glz::meta<obj_t> {
   using T = obj_t;
   static constexpr auto value = object(
      "fixed_object", &T::fixed_object,
      "fixed_name_object", &T::fixed_name_object,
      "another_object", &T::another_object,
      "string_array", &T::string_array,
      "string", &T::string,
      "number", &T::number,
      "boolean", &T::boolean,
      "another_bool", &T::another_bool
   );
};
```

Or using macros:

```c++
GLZ_META(obj_t, fixed_object, fixed_name_object, another_object,
         string_array, string, number, boolean, another_bool);
```

## daw_json_link

```c++
template<>
struct daw::json::json_data_contract<obj_t> {
  using type = json_member_list<
   json_class<"fixed_object", fixed_object_t>,
   json_class<"fixed_name_object", fixed_name_object_t>,
   json_class<"another_object", another_object_t>,
   json_array<"string_array", std::string>,
   json_string<"string", std::string>,
   json_number<"number", double>,
   json_bool<"boolean", bool>,
   json_bool<"another_bool", bool>>;
   
   static constexpr auto to_json_data(const obj_t& v ) {
         return std::forward_as_tuple(v.fixed_object,
                                      v.fixed_name_object,
                                      v.another_object,
                                      v.string_array,
                                      v.string,
                                      v.number,
                                      v.boolean,
                                      v.another_bool);
   }
};
```

## json_struct

```c++
JS_OBJ_EXT(obj_t, fixed_object, fixed_name_object, another_object, string_array, string, number, boolean, another_bool);
```

## Nlohmann JSON

```c++
void to_json(json& j, const obj_t& v) {
   j = json{{"fixed_object", v.fixed_object},
            {"fixed_name_object", v.fixed_name_object},
            {"another_object", v.another_object},
            {"string_array", v.string_array},
            {"string", v.string},
            {"number", v.number},
            {"boolean", v.boolean},
            {"another_bool", v.another_bool}};
}

void from_json(const json& j, obj_t& v) {
   j.at("fixed_object").get_to(v.fixed_object);
   j.at("fixed_name_object").get_to(v.fixed_name_object);
   j.at("another_object").get_to(v.another_object);
   j.at("string_array").get_to(v.string_array);
   j.at("string").get_to(v.string);
   j.at("number").get_to(v.number);
   j.at("boolean").get_to(v.boolean);
   j.at("another_bool").get_to(v.another_bool);
}
```

