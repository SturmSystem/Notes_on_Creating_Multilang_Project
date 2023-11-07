# Notes on binding QML, C++ and Fortran to the one multilang project
This is just some my own notes on binding together multilang project in Qt. Since I gonna use this binding to math implementation, I define three main stages in process of creation of application. Schematically this process may be described as follows
![Main Scheme](Images/Main%20Scheme.png)
So, we have three basic stages. UI design, Math implementation and connection them (management with commands and results). Since first and second stages are hard enough, we should separate them as much as possible to be able to work only on concrete subject. So, the management stage should depend on UI design and Math implementation, not way around. And that's right, because abilities of C++ give us much more flexibility.
Here I gonna make some notes, how do I do these connections. Since I'm not so far in programming I may not understand something, so I'll push here questions for myself to the future.

# Table of Contents
1. [Qt Design Studio and Qt Creator binding](#qt-design-studio-and-qt-creator-binding)
1.1 [Cmake](#11-cmake)
1.2 [Qmake](#12-qmake)
1.3 [QML and C++ interaction](#13-qml-and-c-interaction)
    * [Root Context](#root-context)
    * [Context Object](#context-object)
    * [Context Property](#context-property)
2. [Including Fortran programs to Qt Creator](#2-including-fortran-programs-to-qt-creator)
2.1 [Fortran-C binding](#21-fortran-c-binding)
2.2 [Compilers](#22-compilers)
2.3 [Compile libs](#23-compiling-libs)
2.4 [Include library to Qt lib](#24-include-library-to-qt)


## 1. Qt Design Studio and Qt Creator binding
The main idea nice shown in the video of [DuarteCorporation Tutoriales](https://www.youtube.com/watch?v=9l3peVHccaQ&list=PL54fdmMKYUJvahGcI0cZCwrNesEsNd42V&index=46). He talks in spanish, but still that's understandable.
Let's start with CMake (but futher I'll use qmake).
### 1.1 CMake
1. Firts of all, you should create Design Studio project (let's call it QML). There's qtquickcontrols2.conf file inside the project. You should copy that and paste outside your project.
___In file imports/QML/Constants.qml we should delete or comment strings that contains StudioApplication. Otherwise later it'll cause problems in Qt Creator project. The same thing with .ui.qml files, which should be replaced with .qml files___.
![Creating Design Studio Project](Images/Creating%20Design%20Studio%20project.png) 


2. In Qt Creator let's create Qt Quick Application project, choosing cmake as build system and choosing MSVC and MinGW both as kits.
![Creating Qt Quick](Images/Qt%20Quick%20Application.PNG)
![Choosing Kits](Images/Choosing%20Kits.PNG)

__It is necessary to choose them both if you're gonna build under CMake with MSVC compiler. Otherwise, you'll get the mistake "no CMAKE_CXX_COMPILER could be found"__

__Question:__ Does it work like this under the qmake build system? Why does it work like this under CMake? Maybe MinGW adds some needed flags and options?

3. Add to the project folder recently created Design Studio project folder and qtquickcontrols2.conf file.
![Project folder](Images/Project%20folder.PNG)

4. In Qt Creator project you should create Resource file (.qrc)
___Note, that .qrc file should be added to the same place with main.cpp in CMakeLists___
```
qt_add_executable(appExample
    main.cpp
    QML.qrc
)
```
5. In qrc file you should include the whole created Design Studio project directory (right click on qrc and choosing "Add Existing Directory...") and include qtquickcontrols2.conf file, which you copied earlier (right click on qrc and choosing "Add Existing Files..."). Without separate including of qtquickcontrols2.conf Qt Quicks will work wrong.

6. To make qrc work we need add to CMakeLists option
```
set(CMAKE_AUTORCC ON)
```

7. In the main.cpp we should write the path to your main App.qml file instead of the file, that written there by default.
So, instead of 
```cpp
const QUrl url(u"qrc:/Example/main.qml"_qs);
```
we should write
```cpp
const QUrl url(u"qrc:/QML/content/App.qml"_qs);
```
Since we've done this, our project knows, which qml file to launch. You also may launch build and run, but the result you'll get will be strange. Because the Application doesn't know, where your imported files are placed.

8. To the correct work of importing files, we should also tell to our Application, where the import directory is.
So, we add string
```cpp
engine.addImportPath("qrc:/QML/imports");
```
9. Now, your application works fine and you're able to make changes in Design Studio and Qt Creator separately.

### 1.2 Qmake
Under qmake steps 4-6 are different. Instead of them we just replace some strings in .pro file with string
```
RESOURCES += \
    $$files(QML/*)\
    qtquickcontrols2.conf
```
and 
```
QML_IMPORT_PATH = QML/imports
```


### 1.3 QML and C++ interaction
The question is next: How can we push commands to our Application from Interface and how can we get results from Application to Interface?
This theme is nicely explained by [Lev Alekseevsky](https://www.youtube.com/watch?v=yDbir1zC0wY), he talks in Russian.
Here I'll try to explain it with my own understanding.

What would you do if you'll need to communicate with your friend on the distance? I suppose, you'll give him telephone or transmitter, something that let you talk to him, and him to talk to you. 
The same we do here. Our C++ Application presents to Qml Interface some QObject class (or element of class), which we use to give and push commands.
![C++ and Qml communication](Images/Application%20Interface%20communication.png)

#### Root Context
So, if we wanna our friend to know about transmitter, we should at least give it to him, so, place it to his context. The same thing exists in Qml. It is the QmlRootContext. We are able to deal with it.
Since we created example of QQmlApplicationEngine in ___main.cpp___
```cpp
QQmlApplicationEngine engine;
```
we can get pointer on the rootContext
```cpp
#include <QQmlContext> //Not forget to include this library to deal with context properties
engine.rootContext();
```

#### Context object
First way is to create some class (derived from QObject) and place it to the Root Context.
Let's create the class, named as Transmitter, and create example of this class in ___main.cpp___
```cpp
#include "transmitter.h"
Transmitter appEngine;
```
Then, we can place this object to the root context with
```cpp
engine.rootContext()->setContextObject(&appEngine);
```
Since we done it, we can use Q_INVOKABLE methods, signals and slots of this object in Qml (at every string).


So, we gave our friend the transmitter, but we should teach him, how to use it. But when should we teach him? Of course, when we gave him transmitter.
Qml doesn't know the name of object that we places in Context, so it can just call its methods. So
```js
Connections {
    target: ...
}
```
won't work, because we don't know what is the target.
But there is a way. Let's create somewhere in Qml document the slot, named __onSlot__ (every function in Qml is a slot) and let's our Transmitter class to have a signal __signalName__, which should make slot work.
Then, we may add at our component with needed slot these binding
```js
Component.onCompleted: {
    signalName.connect(onSlot)
}
```
So, when our application will be running and our component will be created, then signal __Transmitter::signalName__ will be connected to __onSlot__.


#### Context property
But such binding of context object and slots isn't really inconvenient to use. Especially if we wanna place more transmitters to Qml context.
So, we can use another way. We may place the concrete object as a Qml context property!
Let's create Transmitter class element in ___main.cpp___
```cpp
#include "transmitter.h"
Transmitter appEngine;
```
Then let's push it to the Qml context
```cpp
engine.rootContext()->setContextProperty("app", &appEngine);
```

Since we've done it, Qml knows that there is the object __app__ in his context.
So, again let's create somewhere in Qml document the slot, named __onSlot__ and let's our Transmitter class to have a signal __signalName__, which should make slot work. 
Then we can bind them like
```js
Connections {
    target: app
    onSignalName:: {
        onSlot()
    }
}
```

## 2. Including Fortran programs to Qt Creator
Fortran language is really cool for math applications, but it still enought fast to compete with C language. So, that's not strange that I've got the idea to use it with Qt Framework.
### 2.1 Fortran-C binding
First of all, Fortran language has binding with C on its own.
To activate this binding, we should say to compiler about it, when we declare procedure, so we use addition like _bind(c)_ and _iso\_c\_binding_ like this
```fortran
subroutine example() bind(c)
    use, intrinsic :: iso_c_binding
    ...
end subroutine example
```

Since we declared function like this, we're able to use it in our C programs. We should [compile object file](#23-compile-libs) and compile our C++ with this object file. In __.cpp__ file we should declare the function we gonna use
```cpp
extern "C" {
    void example();
}
```
This declaration may be replaced at .h file, which we'll include in our C++ file.

The main question is how do we link different types? As we know, Fortran has 5 types, but C has much more and they are not exactly same.
The question is easy. While declaring any variable we should mentions, which kind of type it is. It'll look like this
```fortran
real(kind=c_float) :: var
```
Here we mentioned, that we declare just usual real number, which is the same as C float type. There are tables on how do fortran types and kinds are correlated to C types

|Fortran type | Fortran kind | C type      | 
| :-----------: | :------------: | :----------:  |
|integer      |     c_int    |    int      |
|             |   c_short    | short int   |
| | c_long | long int |
| | c_long_long | long long int |
| | c_signed_int | signed int |
| | c_size_t | size_t |
| | c_int8_t | int8_t |
| | c_int16_t | int16_t |
| | c_int32_t | int32_t |
| | c_int64_t | int64_t |
| real | c_float | float |
| | c_double | double |
| | c_long_double | long double |
| complex | c_float_complex | float_Complex | 
| | c_double_complex | double_Complex |
| | c_long_double_complex | long double_Complex |
| logical | c_bool | bool |
| character | c_char | char |

There are also some procedures that declared in _iso\_c\_binding_ module

| Name | Description |
| :--------: | :-----------------: |
| c_associated | Function to test if a C pointer is associated. |
| c_f_pointer | Function to convert a C pointer to a Fortran pointer. |
| c_f_procpointer | Function to convert a C function pointer to a Fortran procedure pointer. |
| c_funloc | Returns the addres in memory of a C function. |
| c_loc | Returns the address in memory of a C data item. |
| c_sizeof | Returns the size of a C data item in bytes. |

And there are some types to interoperate with C pointers

| Name | Description |
| :--: | :-: |
| c_ptr | Derived type representing any C pointer type. |
| c_funptr | Derived type representing any C function pointer type. |
| c_null_ptr | The value of a null C pointer. |
| c_null_funptr | The value of a null C function pointer. |

And finally it is necessary to remember that in C we're passing arguments by their value, while in Fortran we do it by adress. So, if in Fortran function looks like
```fortran
real(kind = c_double) function do_sum(a,b) bind(c)
    use, intrinsic :: iso_c_binding
    real(kind = c_double) :: a,b
    ...
end function do_sum
```
Then in declaration in C++ it should look like
```cpp
extern "C" {
    double do_sum(double*, double*);
}
```
To link Fortran with Qt I'll create a class, main function of which is to run Fortran written programs. Some kind of interface between Qt C++ and Fortran-C binded program.

### 2.2 Compilers
The question is which compilers to use? 
Basicly, we can derive 2 groups of compilers, that will nicely work with each other

| First group | Second group |
| :-: | :-: |
| gcc, g++, gfortran, PGI | ifort (Intel Compiler), cl (Visual Studio compiler, in Qt it is MSVC compiler) |

PGI compiler is used to add CUDA Fortran. But Intell Compilers have really better optimization. I'll use group of ifort+cl compilers.

### 2.3 Compile libs
Well, let's suppose we created Fortran program with needed function, so now we have to files
```
fort.f90 (with program)
fort.h (with funciton description)
```
So, we create class (let it be named c_f_class), in which we realise calling funcitons. So, we include fort.h in __c\_f\_class.cpp__ (necessary in this file, otherwise later we'll need to add fort.h in our project).
So, we start compiling with
```
ifort /c fort.f90      ==>    fort.obj
```
Next step is to compile class
```
cl /c c_f_class.cpp    ==>    c_f_class.obj
```
And now we create static lib from object files
```
xilib fort.obj c_f_class.obj /out:fort_lib.lib      ==>      fort_lib.lib
```

So, now we have two files:
```
c_f_class.h (with description of c_f_class)
fort_lib.lib (with realization)
```

### 2.4 Include library to Qt

To include our files in Qt Creator project really easy under qmake.
We just should add
```
    INCLUDEPATH += " *absolute path to c_f_class.h* "
    HEADERS += *realtive path to c_f_class.h*/c_f_class.h
    LIBS += *relative path to fort_lib.lib*/fort_lib.lib
```
But! Of course we compiled a class, but compiler will try to find some Fortran libraries. So, we should tell him, where he could find theme. You may start searching some lib (ifmodintr.lib for example) to find, where do they placed on your computer. There will be path like
```
C:/Program Files (x86)/Intel/oneAPI/compiler/2022.2.1/windows/compiler/
```
The last folder 'compiler' includes two needed folders. Firts - the 'include' forlder. In 'include' folder we should choose one of two folders (depending on the 32 or 64 bit your system). I'll choose 'intel64'. 
So, we add this folder to our include paths
```
    INCLUDEPATH += "C:/Program Files (x86)/Intel/oneAPI/compiler/2022.2.1/windows/compiler/include/intel64"
```
Another needed folder is folder with libs realisation. Usually, when you add the path to libs folder, you write
```
    LIBS += -L" *path to libs folder* "
```
Here the same. Needed folder is 'lib/intel64_win'. So, we add
```
    LIBS += -L"C:/Program Files (x86)/Intel/oneAPI/compiler/2022.2.1/windows/compiler/lib/intel64_win"
```

If you did all the things correctly, then your project should be compiled successfully!
__Question:__ How create shared library and include it to Qt?
