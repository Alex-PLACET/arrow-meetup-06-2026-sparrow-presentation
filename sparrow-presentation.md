---
marp: true
lang: en
paginate: true
footer: ![height:20px](resources/github.svg) @Alex-PLACET @JohanMabille @QuantStack

style: |
  section.top {
    display: flex;
    flex-direction: column;
    justify-content: flex-start;
    align-items: flex-start;
    padding-top: 50px;
  }
  
  section {
    display: flex;
    flex-direction: column;
    justify-content: center;
  }
  
  pre {
    font-size: 24px !important;
    line-height: 1.4 !important;
    margin: 10px 0 !important;
    width: 100% !important;
    max-width: 100% !important;
    box-sizing: border-box !important;
    overflow-x: auto !important;
  }
  
  code {
    font-size: 24px !important;
    font-family: Consolas, Monaco, monospace !important;
  }
  
  pre code {
    font-size: 24px !important;
    white-space: pre !important;
  }

  .circle-img-big {
    width: 200px !important;
    height: 200px !important;
    border-radius: 50% !important;
    object-fit: cover !important;
    display: block !important;
    margin: 0 auto !important;
  }
  
  .circle-img {
    width: 100px !important;
    height: 100px !important;
    border-radius: 50% !important;
    object-fit: cover !important;
    display: block !important;
    margin: 0 auto !important;
  }
  
  .invisible-table {
    border: none !important;
    border-collapse: collapse !important;
    background: transparent !important;
    margin: 0 auto !important;
    border-spacing: 0 !important;
    table-layout: fixed !important;
  }

  .invisible-table td {
    border: none !important;
    border-top: none !important;
    border-bottom: none !important;
    border-left: none !important;
    border-right: none !important;
    background: transparent !important;
    padding: 20px !important;
  }
  
  .invisible-table tr {
    border: none !important;
    background: transparent !important;
  }
---

<style>
section::after {
  content: attr(data-marpit-pagination) '/' attr(data-marpit-pagination-total);
}
</style>

![bg opacity:.1 ](resources/cosmo_victory.png)

<h1>The Sparrow ecosystem: Arrow in modern minimal C++20</h1>
<h2>Apache Arrow / Parquet - June 2026 meetup in Paris</h2>
<h4>Alexis Placet<br>
Johan Mabille</h4>
<br>
<br>

![w:500 align-center](resources/logo_scientific_computing.svg)

---
# About

<table class="invisible-table">
    <tr>
        <td>
            <img src="resources/Alexis.png" class="circle-img-big">
            <strong>Alexis Placet</strong></br>
            Software Engineer at QuantStack
        </td>
        <td>
            <img src="resources/Johann.png" class="circle-img-big">
            <strong>Johan Mabille</strong></br>
            Technical Director at QuantStack
            Contributor to Jupyter, mamba, xtensor
        </td>
    </tr>
</table>

---

# Why Sparrow ?

* Apache Arrow's tabular format has become the industry standard
* Apache Arrow is a huge monorepo with many dependencies, while some projects only need the memory format implementation (e.g., ArcticDB)
* Apache Arrow started more than 10 years ago, and C++20 brought new features that we can leverage in a fresh, modern project

---

# What are Sparrow libraries ?

Idiomatic C++20 implementations of the Apache Arrow memory format + canonical extensions, IPC and PyCapsule featuring
* value semantics, ranges, and iterators ...
* Support for both typed and untyped arrays
* Mutability for typed arrays
* Convenient constructors and builders for easy usage
* Built-in std::formatters for all Sparrow types

---

# Sparrow properties

* Zero dependencies for Sparrow
  * ... except when compiling with libc++ (P0355R7 Timezone)
  * ... zstd and lz4 if support for compressed IPC
* Passes all Apache Arrow integration tests, compatible with formats 1.0 to 1.5
* Packaged for Conda, Conan, Fedora and vcpkg
* Licensed under Apache License v2 or BSD

---

# Nullable

```c++
sp::nullable<int> n = 2;
std::cout << n.has_value() << std::endl; // Prints true
std::cout << n.value() << std::endl;     // Prints 2
 
sp::nullable<double> nd = sp::nullval;
std::cout << nd.has_value() << std::endl; // Prints false
```

---

# Typed Array

```c++
sp::primitive_array<int> ar = { 1, 3, 5, 7, 9 };
std::cout << ar.size() << std::endl;  // Prints 4
std::cout << ar.empty() << std::endl; // Prints false
std::cout << pa.front().value() << std::endl; // Prints 1
std::cout << pa.back().value() << std::endl;  // Prints 4
std::cout << pa[2].value() << std::endl;      // Prints 3
try{
  std::cout << pa.at(5).value() << std::endl;   // Throws std::out_of_range
}catch (const std::out_of_range& e) {
  std::cout << "Index out of range" << std::endl;
}
std::for_each(pa.begin(), pa.end(), [](auto n) { std::cout << n.value() << ' '; });
```

---

# C Structures and untyped arrays

```c++
auto [c_arrow_array, c_arrow_schema] = sp::extract_arrow_structures(std::move(ar));
// Use c_arrow_array and c_arrow_schema as you need (serialization, passing them to
// a third party library)
// ...
// You are responsible for releasing the structures at the end
c_arrow_array.release(&c_arrow_array);
c_arrow_schema.release(&c_arrow_schema);
```

---

# C Structures and untyped arrays

```c++
ArrowArray c_arrow_array;
ArrowSchema c_arrow_schema;
tpl::read_arrow_structures(&c_arrow_array, &c_arrow_schema);
sp::array ar(&c_arrow_array, &c_arrow_schema);
// Use ar as you need
// ...
// You are responsible for releasing the structures at the end
c_arrow_array.release(&c_arrow_array);
c_arrow_schema.release(&c_arrow_schema);
```

---
# C Structures and untyped arrays
```c++
sp::array ar(std::move(array), std::move(schema));
// Use ar as you need
// ...
// Do NOT release the C structures at the end; the "ar" variable will do it for you
```

---

# Visit untyped arrays

```c++
ar.visit([]<class T>(const T& typed_ar)
{
    if constexpr (sp::is_primitive_array_v<T>)
    {
        std::for_each(typed_ar.cbegin(), typed_ar.cend(), [](const auto& val)
        {
           ///
        });
    }
    // else if constexpr ...
});
```

---

![bg  fit 65%](resources/sparrow_archi.svg)

---

# Sparrow IPC
```c++
const std::vector<sp::record_batch>& batches = /* ... */;
std::vector<uint8_t> stream_data;
sp_ipc::memory_output_stream stream(stream_data);
sp_ipc::serializer serializer(stream);
serializer << batches << sp_ipc::end_stream;
return stream_data;
```

```c++
std::vector<sp::record_batch> batches = sp_ipc::deserialize_stream(stream_data);
```

---

# Sparrow Rockfinch

## Exporting to python

```c++
PyObject* sparrow_array = sp::pycapsule::create_sparrow_array_object(std::move(my_array));
```

```python
from polars._plr import PySeries
ps = PySeries.from_arrow_c_array(sparrow_array)
```

---

## Exporting to C++

```python
import pyarrow as pa
arrow_array = pa.array([1, 2, None, 4, 5])
sparrow_array = SparrowArray(arrow_array)
```
or 
``` c++
// Receive capsules from Python (e.g., from __arrow_c_array__)
PyObject* schema_capsule = /* ... */;
PyObject* array_capsule = /* ... */;
// Import into sparrow array
sp::array imported_array = sp::pycapsule::import_array_from_capsules(
        schema_capsule, array_capsule);
```

---

# What's Next

* Sparrow tables
* Computation kernels
* GeoArrow

---

# Contributors

<table class="invisible-table" style="table-layout: fixed">
  <tr>
    <td style="text-align: center; width: 200px;">
      <img src="resources/Alexis.png" class="circle-img">
      <strong>Alexis Placet</strong><br>
      @Alex-PLACET
    </td>
    <td style="text-align: center; width: 200px;">
      <img src="resources/Johann.png" class="circle-img">
      <strong>Johan Mabille</strong><br>
      @JohanMabille
    </td>
    <td style="text-align: center; width: 200px;">
      <img src="resources/Thorsten.png" class="circle-img">
      <strong>Thorsten Beier</strong><br>
      @DerThorsten
    </td>
  </tr>
  <tr>
    <td style="text-align: center; width: 200px;">
      <img src="resources/Joël.png" class="circle-img">
      <strong>Joël Lamotte</strong><br>
      @Klaim
    </td>
    <td style="text-align: center; width: 200px;">
      <img src="resources/Julien.png" class="circle-img">
      <strong>Julien Jerphanion</strong><br>
      @jjerphan
    </td>
    <td style="text-align: center; width: 200px;">
      <img src="resources/Hind.png" class="circle-img">
      <strong>Hind Montassif</strong><br>
      @Hind-M
    </td>
  </tr>
</table>

---

# Thanks !

# Resources

- https://github.com/man-group/sparrow
- https://github.com/sparrow-org
