---
title: A Pybind11 Intro based on my Experience Wrapping OpenKarto
layout: post
---

In this post I break down in detail the various challenges of wrapping a `C++` library in `python` using
`pybind11` (you can find the result **pyOpenKarto** [here](https://github.com/safijari/pyOpenKarto)) to get a pleasant python interface. It contains examples of how to do various things which
I had to dig through the documentation to find, so I'm hoping it can help out a weary traveler or two. Note that this is focusing solely on Linux (specifically Ubuntu but that only affects package names).

## Some background on OpenKarto
As the _about_ portion of this website shows, I'm at least part roboticist despite most of my 
time at [Simbe Robotics](http://www.simberobotics.com) being spent on computer vision and machine learning. As such I've had a few tussles with 
algorithms to map an environment using laser scanner data. Most recently I encountered 
(and quite enjoyed working with) [`OpenKarto`](https://github.com/ros-perception/open_karto) which 
was originally developed at SRI and currently maintained by [OSRF](https://www.openrobotics.org/).

## The need for a python wrapper and a solution
In addition to using this library onboard the robot, I realized we could make use of it to post
process some logged laser scans and improve the overall accuracy of our data pipeline. It is here that I hit a small snag: our data pipeline is written in `python`. Now, this isn't _technically_ an issue, what with

1. `python` being designed with [C/C++ extensions](https://docs.python.org/2/extending/extending.html) in mind,
2. `cython` allowing you to [call C++ functions](https://cython.readthedocs.io/en/latest/src/userguide/wrapping_CPlusPlus.html),
3. `Boost.Python` allowing what always appears to be a [simple way to interface](https://www.boost.org/doc/libs/1_68_0/libs/python/doc/html/index.html) `python` with `C++` (if you can get it to work...)
4. and `pybind11` allowing for a very `Boost.Python` like interface but no `boost` requirement ... also it's header only!!!

Number 4 was brand new to me. I had tried (and mostly struck out, repeatedly) with all the other options. So I gave it a try. I started with the excellent [python example](https://github.com/pybind/python_example) since that allows me to build/deploy with the `setup.py` file.

## Dealing with external dependencies
This is the first thing I needed to do a bit of digging to find as the example does not tackle it.
Building OpenKarto needs at least `libeigen3-dev` and `libboost-thread-dev` to function correctly (darn, 
I couldn't avoid boost entirely). The former is header only, the latter is dynamically linked. Note that this means that `libboost-thread` must be present on the target system unless you release a pre-packaged python wheel using AuditWheel (which I will explain in a future post).

All of this needs to be specified in the `Extension` module for my library as shown here:

```python
ext_modules = [
    Extension(
        'openkarto',      # python module name
        ['src/PythonInterface.cpp', 
        'src/Mapper.cpp', 
        'src/Karto.cpp'], # All the cpp files that need to be built
        include_dirs=[
            'include',    # OpenKarto's necessary .h files
            '/usr/include/eigen3/', # Eigen 3's headers
            get_pybind_include(),
            get_pybind_include(user=True)
        ],
        libraries=['boost_thread'], # Tell the compiler to find libboost_thread
        library_dirs=['/usr/lib/'], # Unnecessary in this specific case
        language='c++'
    ),
]
```

As you can see the overall changes are minimal and pretty straight forward. Note that for most correctly
installed libraries on linux (`libboost-thread` included) you should not need to specify a library dir
as that is already searched.

## The interface file

This file, that I called [`PythonInterface.cpp`](https://github.com/safijari/pyOpenKarto/blob/python-devel/src/PythonInterface.cpp), is the only addition I needed to make (other than the `setup.py`)
to the base `open_karto` repo. To allow for more overall control, I abstracted the details of using
the karto api into a `MapperWrapper` class which takes care of things like setup and teardown
as well as reduce some general tedium. To expose this class, I needed to add the following code:

```python
#include <pybind11/pybind11.h>
#include <pybind11/stl.h>

PYBIND11_MODULE(openkarto, m) {
  py::class_<MapperWrapper>(m, "MapperWrapper")
    .def(py::init<std::string, double, double, double>())
    .def("reset", &MapperWrapper::Reset)
    .def("process_scan", &MapperWrapper::ProcessLocalizedRangeScan)
    .def("get_processed_scans", &MapperWrapper::GetProcessedScans, py::return_value_policy::reference)
    .def("create_occupancy_grid", &MapperWrapper::CreateOccupancyGrid)
    .def_property_readonly("name", &MapperWrapper::getName)
    .def_property_readonly("range_finder", &MapperWrapper::getRangeFinder)
    .def_property_readonly("mapper", &MapperWrapper::getMapper, py::return_value_policy::reference);

  m.def("create_custom_rangefinder", &CreateCustomRangeFinder);
}
```

Note that this was followed by more code I'm not showing here which allows some members of this class to be accessed. 
The very first line defines the module itself. `openkarto` is the name of the 
exported module and `m` is a variable that we can use to modify the module and add
functions and classes to it.

### Exposing a class
To expose a `C++` class, you need at least a minimum of

```c++
py::class_<ClassName>(moduleObject, "ClassNameInPython");
```

In the above example the template argument to `py::class_` is `MapperWrapper`, the
class I want to export. Note that I'm choosing to call it by the same name in python though
that certainly can be different.

This by itself wouldn't be very useful without exposed fields/functions/properties/etc.
To do this, you can chain methods at the end of that statement. Perhaps the first method
we need to add would be an `init` so we can create this class on the python side.

### Constructors/Initializers
Just chain the following to your class defintions:
```c++
.def(py::init<arg1_type, arg2_type, ...>())
```
where the template arguments can number in 0, as the case may be. Note that this exposes the
`C++` class's constructor. In some cases you might want to expose a factory method instead.
This is also very straight forward and the example below from another class in karto should make
it clear:

```c++
py::class_<karto::LaserRangeFinder>(m, "LaserRangeFinder")
  .def(py::init(&karto::LaserRangeFinder::CreateLaserRangeFinder));
```

Note that it's perfectly fine (as you'll soon see) to expose a class _without_ an initializer
if the class is never intended to be instantiated on the `python` side. The `OccupancyGrid` class
in `karto` is always created on the `C++` side so it has no initializer.

### Normal methods
Exposing a normal method of the class is perhaps most straight forward:
```c++
  .def("python_func_name", &ClassName::MethodName);
```
Note that there is no need to define arguments/their types as well as return types.
`pybind11` will automatically take care of this. Note however that a returned `C++` type
**must** be:

1. translatable to python (true for most primitive types and stl containers) or
2. a python type constructed on the `C++` side (e.g. `py::make_tuple(1234, "hello");`) or
3. exposed to python.

If one of these three conditions is not met, trying to access the returned value in `python`
will throw a runtime exception about the type not being known.

Note that in some cases you might need to mess with the `return_value_policy` (more info [here](https://pybind11.readthedocs.io/en/stable/advanced/functions.html)). This is a topic I don't fully
understand but at least in one case I needed to ensure that the python side does not take ownership of the object so I could avoid some double deletes, and this was done by setting the return policy
to `reference`.

```c++
.def("get_processed_scans", &MapperWrapper::GetProcessedScans, 
   py::return_value_policy::reference)
```

### Properties and public fields
Now we get to one of my favorite things, mapping getters and setters to properties. There are
two ways to do this and the distinction is obvious.

```c++
.def_property("property_name", &ClassName::Getter, &ClassName::Setter)
```

or

```c++
.def_property_readonly("property_name", &ClassName::Getter)
```

Note that pybind11 has support for C++ lambdas in case you're wanting to expose a field
that doesn't already have a getter/setter, you can do something like this:

```c++
.def_property_readonly("property_name", [](const ClassName &a) {
    return a.field;
})
```

This can be done in all the presented cases, not just for properties. Finally, you can 
expose a public field directly if you choose to

```c++
.def_readwrite("python_field_name", &ClassName::fieldName)
```
### Printing the object on the python side
There are few things in `python` land worse than

```python
In [42]: print(obj)
Out[42]: <module.Confusion at 0x12345678AB>
```

Thankfully you can simpley define a `__repr__` for your class on the `C++` side same
as you would have on the `python` side. In karto's case, 
it was important to be able to print out the data in the 2D pose class so I could keep my sanity.
Note that we are returning a `C++` string which gets automatically translated to a python
string.

```c++
.def("__repr__", [](const karto::Pose2 &a) {
  std::stringstream buffer;
  buffer << "(x: " << a.GetX() << ", y: " << a.GetY() << ", heading: " << a.GetHeading() << ")\n";
  return buffer.str();
})
```

### Normal functions
This is the same as defining a method for the class except the `.def` is called on the module
itself. As an example, I have a factory method for creating a concrete instance of a templated 
class, and I can expose it like so:

```c++
m.def("create_custom_rangefinder", &CreateCustomRangeFinder);
```

### Enums
Karto defines the type of laser range finder used as well as states of each
cell of a grid map as enums. To use these from the python side I needed to expose them
like so:

```c++
py::enum_<karto::GridStates>(m, "GridStates")
  .value("Unknown", karto::GridStates::GridStates_Unknown)
  .value("Occupied", karto::GridStates::GridStates_Occupied)
  .value("Free", karto::GridStates::GridStates_Free);
```

The above maps `GridStates::GridStates_Unknown` to `GridStates.Unknown` 
on the python side.

## Building, installing, and packaging
You can build the package using the `setup.py` file 
```
python setup.py build
```
which will create a new build directory and place one or more `.so` files inside it representing
your module. To import the module, you can run python in that directory and then try to import
the module name. If everything was build/linked correctly you should be able to import and use
your shiny new module. This isn't a particularly good way to develop, however, and a better solution
might be to install the pacakge in dev mode in your `python` environment. Do this using
```
pip install -e .
```
The additional `-e` does an "editable" install. All that means is that instead of your files
getting copied somewhere in a `site-packages` directory, `pip` has made symlinks to them.
This only has to be done once and afterwards the `build` command can be used to reflect any changes
on the `C++` side. Note that you can omit the editable flag but that would cause a proper install
of the package, and no changes will be reflected in your python environment until you do the install again.

Packaging is a bit more nuanced given this is not a pure python module. I'll only summarize it here and explain the best option at length in a future post. Other than simply leaving the code on github with some instructions to build from source, you have 4 options:

1. `python setup.py sdist`: Perhaps the worst option as it's basically the same as telling someone to build from source. It will take all your source code and `setup.py` file, and `tar` it up with your version number (and put it in the `dist` directory). Anyone installing it will need to have at least `pybind11` installed as well as the `devel` versions of any of your `C++` code's external dependencies (`libeigen3-dev` and `liboost-thread-dev` for `karto`).

2. `python setup.py bdist_wheel`: This requires that you have the `wheel` package installed. It's a somewhat better option than the last one since no compilation will be required on the user's part. The command will create a _wheel_ which is a _binary_ `python` package (it's a zip file that is allowed to contain `.so` and other binary files). They will, however, need either the `dev` or *normal* (if any) version of your `C++` code's dependencies and version clashes are entirely possible. This option is perfect when your `C++` code has no external dependencies but that is typically not going to be the case.

3. [`auditwheel`](https://github.com/pypa/auditwheel): This tool, sanctioned by the Python packaging authority, uses another nifty little tool called [`patchelf`](https://nixos.org/patchelf.html) to find all the `.so` files that your module needs to be linked against and puts them inside an existing wheel (which you can make using the previous method). This is a much better option but depending on where you build the wheel and where you run the `auditwheel` command, it can have bad effects when deploying your package to a different platform (e.g. if I run `auditwheel` on Ubuntu 18.04 and then try to run the resulting package on 16.04, it straight up will refuse to import because of conflicting `glibc`).

4. `auditwheel` + [`manylinux`](https://github.com/pypa/manylinux): [PEP 513](https://www.python.org/dev/peps/pep-0513/) defines how to create maximally compatible and self contained python wheels, and `manylinux` is a build environment (they supply `docker` containers!). If you install all your dependencies in a `manylinux` `docker` container and then build and patch the wheel inside it, it will most likely run on any linux version from after 2007. This is much easier said than done, in fact it can be downright horrifying for some dependencies. I will spend another article describing this process for karto.

## The Final Product and Conclusion

The original `open_karto` repo has a sample program that creates two circular laser scans processes
them, below I show `python` code doing the same thing. Note that the `process_scan` method expects a `vector<float>`
on the `C++` side but I can pass a normal `python` list. All the conversions are taken care of.

![example](/static/images/pyOpenkarto_example.png)

All in all this was a great success for me. Not only does it allow me to easily integrate karto with my python codebase, 
it also allows me to finetune the parameters and experiment in a notebook, which is a huge boon to productivity.

I was pleasently surprised by how straight forward it was to get this all done with `pybind11`. It's a fantastic effort
and I will be using it a lot more (I've already wrapped two other codebases for work). I hope lessons learned during my 
journey proves helpful to you as well. Please use the disqus section below for any questions/comments.
