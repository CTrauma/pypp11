# Porting Py++ to generate [pybind11] code

The following list describes all changes applied to a file generated by Py++,
which made the code work with [pybind11]. Therefore, it is a list of changes in
the output of Py++, hinting towards, but not strictly specifying, where changes
have to be made in the Py++ internals.

Note, however, that this listing may be incomplete, so feel free to add required 
changes if you find them.

## General

* [ ] `#include "boost/python.hpp"` -> `#include <pybind11/pybind11.h>`

* [ ] Added `#include <pybind11/operators.h>` at top of file to support operator
  overloading.

	**Note**: This is only needed if operator overloading is actually present.

* [x] `namespace bp = boost::python;` -> `namespace py = pybind11;`

* [x] `bp::` -> `py::`

* [ ] `BOOST_PYTHON_MODULE` -> `PYBIND11_PLUGIN`

* [ ] Added at beginning: `py::module m("MODULE_NAME", "MODULE_DESCRIPTION");`

	**MODULE_NAME** [ ] - name used to import the module in python: 
				  `import MODULE_NAME`. This has to match the module name in
				  `PYBIND11_PLUGIN(MODULE_NAME)`.

	**MODULE_DESCRIPTION** [ ] - short description for the module.

	Both values need to be provided by the user through the Py++11 interface,
	so it may be required to add those two as parameters for the appropriate
	functions.

* [ ] Added at end: `return m.ptr();`

* [ ] Replaced statements of the following pattern:

    
    ```
    m.attr("NAME") = class_instance;
    ```

    with

    ```
    m.attr("NAME") = py::cast(class_instance);
    ```

## Classes and Enumerations

* [ ] Replaced statements of the following pattern:

    ```
    bp::class_<ClassName>("ClassName")
    ```
    
    with

    ```
    py::class_<ClassName>(m, "ClassName")
    ```

* [x] Replaced statements of the following pattern:

    ```
    bp::enum_<EnumName>("EnumName")
    ```
    
    with

    ```
    py::enum_<EnumName>(m, "EnumName")
    ```

* [ ] Replaced statements of the following pattern:

    ```
    typedef bp::class_<ClassName> ClassName_exposer_t;
    ClassName_exposer_t ClassName_exposer = ClassName_exposer_t("ClassName", "Documentation", bp::init<>());
    ```
    
    with

    ```
    py::class_<ClassName> class_name(m, "ClassName", "Documentation", py::init<>());
    ```

* [ ] Removed statements of the pattern: `bp::scope ClassName_scope(ClassName_exposer);`

* [ ] Replaced statements of the following pattern:

    ```
    bp::enum_<ClassName::EnumName>("EnumName")
    ```

    with


    ```
    py::enum_<ClassName::EnumName>(class_name, "EnumName")
    ```

* [ ] Replaced statements of the following pattern:

    ```
    ClassName_exposer.def()
    ```

    with


    ```
    class_name.def()
    ```

* [ ] Removed `{}` (local scopes) where they were unnecessary

* [ ] Replaced statements of the following pattern:

    ```
    class_name.alias(ClassName)
    ```

    with

    ```
    class_name.alias<ClassName>()
    ```

* [ ] Replaced statements of the following pattern:

    ```
    bp::class_<ClassName, bp::bases<ParentClass>>("ClassName", "Documentation")
    ```

    with

    ```
    py::class_<ParentClass> parent_class(m, "ParentClass")
    py::class_<ClassName>(m, "ClassName", parent_class, "Documentation")
    ```

	**Note**: The same thing could have been achieved using the `py::base<>` 
	class:

	
    ```
    py::class_<ClassName>(m, "ClassName", py::base<ParentClass>(), "Documentation")
    ```

	With this approach, there is no need to explicitly name the `py::class_` 
	object.

* [ ] Removed `bp::no_init` from wrapper definitions

* [ ] Replaced statements of the following pattern:

    ```
    py::class_<ClassName> class_name("ClassName")
        .def(py::init<>());
    ```

    with

    ```
    py::class_<ClassName> class_name("ClassName");
        
    class_name.def(py::init<>());
    ```

    **Note**: The former case would not compile. This is only needed when a 
	`py::class_` object is named, instead of directly exposing it.

* [ ] Replaced statements of the following pattern:

    ```
    py::implicitly_convertible<const ClassName&, ClassNameOne>();
    ```

    with

    ```
    py::implicitly_convertible<ClassName, ClassNameOne>();
    ```

    **Note**: Generally speaking, every accessor needs to be removed within 
    `py::implicitly_convertible<>()`


## Functions

* [ ] Replaced function pointer definition of the pattern:

    ```
    (return_type ( ClassName* [ ] )(type arg))(&ClassName::function_name))
    ```

	with

    ```
    &ClassName::function_name
    ```

    where the former was not strictly necessary, for example to determine the
    exact version of the function when it is overloaded.
    
* [ ] There seems to be missing a pybind11-counterpart to `boost::noncopyable`, so
  replaced statements of the following pattern:

    ```
    bp::class_<ClassName, boost::noncopyable>("ClassName", "Documentation")
    ```

    with


    ```
    py::class_<ClassName>(m, "ClassName", "Documentation")
    ```

	**Note**: Is this in need to be fixed or is a C++ inheritance
	solution made by the library itself sufficient?

* [ ] Removed statements of the following pattern: `bp::optional< ... >`

* [ ] Replaced `def_readonly` with `def_readonly_static` and `def_readwrite` with
  `def_readwrite_static` where the type was actually `static const` or just 
  `static`, respectively.

* [ ] Commented out every member function that would have been exposed by [pybind11]
  code, but is protected.

	**Note**: This might have been an error by Py++, so the source has to be checked
	to make sure this is not happening. It might also just be a difference in
	[pybind11] and [Boost.Python][boost_python].

* [ ] Replaced statements of the following pattern:

    ```
    py::class_<ClassName> class_name("ClassName")
        .staticmethod("method_name");
    ```

    with

    ```
    py::class_<ClassName> class_name("ClassName")
        .def_static("method_name", &ClassName::method_name, py::arg("arg"), "Documentation");
    ```

* [ ] Added exact function pointer definitions in statements of the following
  pattern:

    
    ```
    .def("function_name",
        , &ClassName::function_name)
    ```

    becomes

    ```
    .def("function_name",
        , (return_type (ClassName::*)(type arg)) &ClassName::function_name)
    ```

    where "function_name" had overloads.

### Constructors

* [ ] Moved `py::init<>()` functions previously inside of `py::class_` constructors
  into their own `py::class_::def()` call:

    ```
    bp::class_<MyClass>("MyClass", bp::init<>());
    ```

    becomes

    ```
    py::class_<MyClass>(m, "MyClass")
        .def(py::init<>());
    ```

    **Note**: This is due to [pybind11] handling keyword arguments, default values
    and documentation strings in constructor definitions differently than 
	[Boost.Python][boost_python] does. The exact explanation will follow, but 
	basically, without this step, the above features would not be possible.

* [ ] Moved definition of keyword arguments for constructors using `py::arg()`
  outside the `py::init<>()` function, into the outer `py::class_::def()`
  function:

    ```
    py::class_<MyClass>(m, "MyClass")
        .def(py::init<int>(py::arg("my_integer")));
    ```

    becomes

    ```
    py::class_<MyClass>(m, "MyClass")
        .def(py::init<int>(), py::arg("my_integer"));
    ```

    **Note**: Without this step, keyword arguments would not be possible. This is
    due to the fact that the `py::init<>()` function does not take any
    arguments and [pybind11] exposes constructors almost like normal functions,
    only omitting the implicit "__init__" string as name and passing 
    `py::init<>()` instead of a function pointer.

* [ ] Moved definition of default values for constructors parameters using 
  `py::arg()` outside the `py::init<>()` function, into the outer 
  `py::class_::def()` function:

    ```
    py::class_<MyClass>(m, "MyClass")
        .def(py::init<int>(py::arg("my_integer") = 1729));
    ```

    becomes

    ```
    py::class_<MyClass>(m, "MyClass")
        .def(py::init<int>(), py::arg("my_integer") = 1729);
    ```

    **Note**: Without this step, default values would not be possible. This is
    due to the fact that the `py::init<>()` function does not take any
    arguments and [pybind11] exposes constructors almost like normal functions,
    only omitting the implicit "__init__" string as name and passing 
    `py::init<>()` instead of a function pointer.

* [ ] Moved documentation strings for constructors outside the `py::init<>()` 
  function, into the outer `py::class_::def()` function:

    ```
    py::class_<MyClass>(m, "MyClass")
        .def(py::init<>("Default constructor."));
    ```

    becomes

    ```
    py::class_<MyClass>(m, "MyClass")
        .def(py::init<>(), "Default constructor");
    ```

    **Note**: Without this step, documentation strings would not be possible. This 
    is due to the fact that the `py::init<>()` function does not take any
    arguments and [pybind11] exposes constructors almost like normal functions,
    only omitting the implicit "__init__" string as name and passing 
    `py::init<>()` instead of a function pointer.

### Operator Overloading

* [ ] Replaced statements of the following pattern:


    ```
    .def(py::self * [ ] py::other<ClassName>())
    ```

    with

    ```
    .def(py::self * [ ] ClassName())
    ```

    where the "*" operator could be any operator.

### Return-Value-Policies

* [ ] Replaced statements of the following pattern:

    ```
    bp::return_value_policy< py::copy_const_reference >()
    ```

    with

    ```
    py::return_value_policy::copy
    ```

    for functions that return a constant reference, plus, moved that after the
    documentation string.

* [ ] `py::return_self<>()` -> `py::return_value_policy::reference_internal`

* [ ] Placed `py::return_value_policy::copy` as return value policy where the return
  type matched the pattern `const ClassName&`.

* [ ] Placed `py::return_value_policy::reference_internal` as return value policy
  where the return type matched the pattern `ClassName&`.

### Virtuality

* [ ] Replaced [Boost.Python][boost_python] style wrappers for (pure) virtual 
  functions with [pybind11] style wrappers:

    Note that the changes are based on [example 12][pybind11_example12] from
    the [pybind11] repository.

    * [ ] Replaced wrapper definition:

        ```
        struct ClassName_wrapper : ClassName {
        ```

        with

        ```
        class ClassNameWrapper : public ClassName {
        public:
        ```

    * [ ] Replaced constructor definition (now inheriting the constructors):

        ```
        ClassName_wrapper()
        : ClassName()
          , bp::wrapper< ClassName >(){
            // null constructor
        
        }
        ```

        with

        ```
        using ClassName::ClassName;
        ```

    * [ ] Replaced virtual function wrapper:

        ```
        virtual return_type function( arg_type argument) modifier {
            if( py::override func_function = this->get_override( "function" ) )
                return func_function( index );
            else{
                return this->ClassName::function( argument );
            }
        }
        ```

        with

        ```
        virtual return_type function( arg_type argument) modifier {
            PYBIND11_OVERLOAD(
                return_type,
                ClassName,
                function,
                argument /*,
                argument1,
                argument2,
                ...*/
            );
        }       
        ```

    * [ ] Removed generated default functions:

        ```
        return_type default_function( type argument ) modifier {
                return ClassName::function( argument );
        }
        ```

    * [ ] Replaced pure virtual function wrapper:

        ```
        virtual return_type function( arg_type argument) modifier {
            bp::override func_function = this->get_override( "function" );
            func_function( argument );

        }
        ```

        with

        ```
        virtual return_type function( arg_type argument) modifier {
            PYBIND11_OVERLOAD_PURE(
                return_type,
                ClassName,
                function,
                argument /*,
                argument1,
                argument2,
                ...*/
            );
        }       
        ```

    * [ ] Replaced `class_.def()` call:

        ```
        bp::class_< ClassName_wrapper >("ClassName", "Documentation")
            .def("function", &ClassName_wrapper::function, "Documentation");
        ```
        
        with

        ```
        py::class_<ClassNameWrapper>(m, "ClassName", "Documentation")
            .def("function", &ClassName::function, "Documentation");
        ```

* [ ] Replaced statements of the following patter:

    ```
    &ClassName_wrapper::function
    ```

    with

    ```
    &ClassName::function
    ```

    within the `class_.def()` function calls.

* [ ] Removed statements of the following pattern: `bp::pure_virtual(&ClassName::function)`

<!-- Maybe avoidable through settings of library
* [ ] Replaced statements of the following pattern:

    ```
    #include "header"
    ```

    with

    ```
    #include <header>
    ```
-->

<!-- References -->
[boost_python]: http://www.boost.org/doc/libs/1_60_0/libs/python/doc/html/index.html "Boost.Python"
[pybind11]: http://pybind11.readthedocs.org/en/latest/index.html "pybind11"
[pybind11_example12]: https://github.com/wjakob/pybind11/blob/master/example/example12.cpp "pybind11 - example12"
[pypp]: http://sourceforge.net/projects/pygccxml/ "Pyplusplus"
