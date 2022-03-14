# effectiveModernCpp_rlvaluereferences_movesemantic_forward
 L, R value and Universal references, move semantic and forwarding Notes and Code Snippets from Effective Modern C++ by Scott Meyers

```cpp

#include <iostream>
#include <vector>
#include <string>

using namespace std;

/* 
    NOTES
        All paramaters are lvalues. 
            Don't forget the difference between parameter and argument. (params are lvalues, where argument can be lvalue or rvalue)
        All what std::moves does is casting the parameter unconditionally to rvalue, where std::forward can pass lvalue argument of a functions 
            as lvalue to another function but rvalues to rvalues in the case where the arguments which passed in the function are rvalues.
        Reference Collapsing is deduction of a type as lvalue when either of the param and argument is lvalue. otherwise(both is rvalue) is a rvalue
            for template T&&, T will be deduced as T& when argument is a lvalue(so Widget& && ==> Widget&)
                              but for a argument which is a rvalue, T will be as non-refence T. (Widget && ==> Widget&&)
        In templates T is deduced with T& for arguments which are lvalues, as T can be deduced as non-reference T for rvalues.
            template<typename T>
            void f(T&& param);

            Widget w; // lvalue
            Widget x(); // func x returns a rvalue
            f(w); // T is deduced as Widget&. so a lvalue reference. (due to reference collapsing. Widget& && param ---> Widget&)
            f(std::move(w)); // std::moves(w) is an rvalue. so T is Widget

        Rvalue-references and Universal references are not the same. For universal reference there must be a type-deduction and only a form of T&&.
            const T&& param is not a universal reference. rather refers to a rvalue. (even adding a key 'const' breaks the universal rule.)
        
        Use std::move on rvalue references, but std::forward on universal references.
            template<typename T>
            void f(T&& newName){ name = std::forward<T>(newName); }
            // using of std::move above is wrong. it will cause that local variable which is passed in will be unknown. (see below)

            std::string x = "hey";
            f(x);
            // hier x is unknown. (in the case where std::move is being used in the function f and newName is a universal reference.)
            
        Do the same thing above for rvalue references and universal references being returned from functions that return by value
            template<typename T>
            Fraction reduceAndCopy(T&& frac) // frac is universal reference
            {
                return std::forward<T>(frac); // move rvalue into return value, but copy if it is lvalue
            }

            Matrix operator+(Matrix&& lhs, const Matrix& rhs){
                lhs += rhs;
                return std::move(lhs); // move lhs into return value
            }

        Never apply std::move or std::forward to local objects if they would be otherwise eligible for the return value optimization.
            template<typename T>
            Widget makeWidget(){
                Widget w;
                return std::move(w); // WROOOOOOOOOOONG!!!!!!!!!!!!!! or std::forward....
            } 
            
            template<typename T>
            Widget makeWidget(Widget w){
                return std::move(w); // WROOOOOOOOOOONG!!!!!!!!!!!!!! or std::forward....
            } 
*/

// implementing of std::move( it will cast unconditionally to rvalue reference and return it.)
template<typename T>
decltype(auto) move(T&& param) // typename std::remove_reference<T>::type&&
{
    using ReturnType = std::remove_reference_t<T>&&; // std::remove_reference<T>::type&&
    return static_cast<ReturnType>(param);
}

// implementing of std::forward
/* 
    for lvalue(T is Widget&): 
        Widget& && forward(Widget& param) // Widget& && will be Widget&
        { 
            return static_cast<Widget& &&>(param); // Widget& && will be Widget&
        }

        // also lvalue reference will be returned
    
    for rvalue(T will be non-reference also Widget)
        Widget&& forward(Widget& param)
        {
            return static_cast<Widget&&>(param); // lvalue param (Widget&) will be casted to rvalue reference and returned.
        }
*/
template<typename T>
T&& forward(std::remove_reference_t<T>& param) 
{
    return static_cast<T&&>(param);
}

/*
    Avoid overloading on universal references
        std::multiset<std::string> names; // global data structure
        // bad version
        void logAndAdd(const std::string& name){
            auto now = std::chrono::system_clock::now();
            log(now, "logAndAdd");

            names.emplace(name);
        }

        std::string petName("Darla");
        logAndAdd(petName); // pass lvalue std::string (name will be copied into names)
        logAndAdd(std::string("Persephone")); // pass rvalue std::string (even though arg is rvalue, name itself is lvalue so it will be copied.)
        logAndAdd("Patty Dog"); // pass string literal (this will copied too. even here there is another problem. no need to create temporary std::string for "Patty Dog" 
                                                        at all. Emplace would have used the string literal to create std::string object directly inside the std::multiset.)

        // better version with universal refence

        template<typename T>
        void logAndAdd(T&& name){
            auto now = std::chrono::system_clock::now();
            log(now, "logAndAdd");
            names.emplace(std::forward<T>(name));
        }

        // Things to remember
            Overloading on universal references almost lead to the universal reference overloading being called more frequently than expected.
                think of the example above. create a overload function that takes integer as parameter.

                std::string nameFromIdx(int idx);

                void logAndAdd(int idx){
                    auto now = std::chrono::system_clock::now();
                    log(now, "logAndAdd");
                    names.emplace(nameFromIdx(idx));
                }

                logAndAdd(22); // calls int overload 

                short nameIdx = 25;
                logAndAdd(nameIdx); // calls universal reference overload. not the int one. and it won't compile. (because std::string doesnt have a short overload.)
*/

/* 
    How to combine universal references and overloading without the problems.

*/

std::string nameFromIdx(int idx)
{
    return std::string("alp");
}

template<typename T>
void logAndAdd(T&& name)
{
    std::cout << "universal references, name = " << name << "\n";
}

void logAndAdd(int idx)
{
    std::cout << "int overload, name = " << nameFromIdx(idx) << "\n";
}

// better version for combinin universal ref and overloading (TAG DISPATCH, also like std::false_type, std::true_type)

template<typename T>
void logAndAddImpl(T&& name, std::false_type)
{
    std::cout << "logAndAddImpl, universal \n";
}

void logAndAddImpl(int idx, std::true_type)
{
    std::cout << "logAndAddImpl, int overload \n";
}

template<typename T>
void logAndAddV2(T&& name)
{
    logAndAddImpl(std::forward<T>(name), std::is_integral<std::remove_reference_t<T>>());
}

// out of topic: implement string_format. NOTE: use std::format in c++20
#include <memory>
#include <string>
#include <stdexcept>

template<typename ... Args>
std::string string_format( const std::string& format, Args ... args )
{
    int size_s = std::snprintf( nullptr, 0, format.c_str(), args ... ) + 1; // Extra space for '\0'
    if( size_s <= 0 ){ throw std::runtime_error( "Error during formatting." ); }
    auto size = static_cast<size_t>( size_s );
    auto buf = std::make_unique<char[]>( size );
    std::snprintf( buf.get(), size, format.c_str(), args ... );
    return std::string( buf.get(), buf.get() + size - 1 ); // We don't want the '\0' inside
}

/* 
    Item 27: Familiarize yourself with alternatives to overloading on universal references.
    Create a class Person that has copy, move operations and a constructor with universal reference when argument being passed is not Person or doesn't inherit it. 
    Use enable_if for that universal ctor. (And when a class derived from Person is being passed into the Person, universal ctor muss be still not called.)
*/

class Person{
    public:
        template<typename T, typename = typename std::enable_if_t<
                                                        !std::is_base_of<Person, std::decay_t<T>>::value &&
                                                        !std::is_integral<std::remove_reference_t<T>>::value
                                                    >>
        Person(T&& n) : name(std::forward<T>(n))
        {
            static_assert(std::is_constructible<std::string, T>::value, "Parameter n can't be used to construct a std::string");
            std::cout << "universal ctor is called\n";
        }

        Person(Person&&) = default; // equals Person(Person&& p) : name{std::move(p.name)} {}
        Person& operator=(Person&&) = default;

        Person(const Person&) = default;
        Person& operator=(const Person&) = default;

        const std::string& GetName() { return name; }

    private:
        std::string name;
};

/* 
    Item 30: Familiarize yourself with perfect forwarding failure cases.
*/
void f(const std::vector<int>& v)
{
    for (auto const& i : v){
        std::cout << "from func f: " << i << "\n";
    }
}

template<typename T>
void fwd(T&& param)
{
    f(std::forward<T>(param));
}

template<typename... T>
void fwd(T&&... params)
{
    f(std::forward<T>(params)...);
}

int main()
{
    logAndAdd(std::string("Hey"));
    logAndAdd(12); // calls int overload
    short idx = 23;
    logAndAdd(idx); // calls universal overload!!! not the int one!!!! to solve this see logAndAddV2

    logAndAddV2(std::string("Selam"));
    logAndAddV2(12); // calls int overload
    logAndAddV2(idx); // calls int overload too! so short will be recognized from int overload. the problem is solved.
    
    std::cout << string_format("lan %s", "selam") << "\n";

    Person p1 = "P1"; // universal ctor will be called
    Person p2 = p1; // lvalue copy assignment will be called.
    std::cout << "name of p1: " << p1.GetName() << " \n";
    std::cout << "name of p2: " << p2.GetName() << " \n";
    Person p3 = std::move(p2); // rvalue copy assignment
    std::cout << "name of p3: " << p3.GetName() << " \n";

    f({1,2,3});
    //fwd({4,5,6}); // error! couldn't deduce template parameter T!!!
    auto x = {4,5,6}; // x deduces as std::initializer_list
    fwd(x);
}


```