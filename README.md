# units <a href="https://travis-ci.org/nholthaus/units" target="_blank"><img src="https://travis-ci.org/nholthaus/units.svg?branch=master"/></a> <a href="https://ci.appveyor.com/project/nholthaus/units" target="_blank"><img src="https://ci.appveyor.com/api/projects/status/github/nholthaus/units?svg=true&branch=master"/></a>
a compile-time, header-only, unit conversion library built on c++14 with no dependencies.

Latest Release - v2.0.3
--------

New features:
 - `unit_t` types are now trivial types.
 -  `unit_t` types support the unary minus (negation) operator.
 - Compile-time unit arithmetic via `unit_value_t`.
 - Unit-enabled ports of most `<cmath>` functions, including c++11 extensions.
 - Square-root manipulators for `unit`, `unit_t`, and `unit_value_t`.
 - Improved documentation.

Tested on:
 - gcc -4.9
 - msvc2013
 - msvc2015

<a href="https://github.com/nholthaus/units/releases/tag/v2.0.3" target="_blank">Download units v2.0.3</a>

Documentation
-------------

The full documentation is available *[here](http://nholthaus.github.io/units)*.

Description
-----------

The library consists of a single file (include/units.h), plus unit tests. To incorporate the library into your project, simply copy the header into a location in your include path. A CMake project is included to build the unit tests and documentation if desired.

The library provides a set of types, containers, and traits to solve dimensional analysis problems, that is, problems involving dimensioned physical quantities. The conversions between units are defined as ratios at compile time, making the library _incredibly_ fast. Additionally, specifying units as _types_, rather than variable suffixes (or not at all), provides complete type-safety within the compiler. This means that code that accidentally misuses units or which has errors in the dimensional analysis _will fail at compile-time, not at run-time_. 

The unit test file `unitTests/main.cpp` contains example usage of every type, trait, and function contained in the library, and while not exactly user-friendly, can be a valuable resource.

Unit tags
---------

Unit tags are the foundation of the unit library. Unit tags are types which are never instantiated in user code, but which provide the meta-information about different units, including how to convert between them, and how to determine their compatibility for conversion.

All unit tags are defined in namespaces under the `units` namespace, such as `units::length` or `units::angle`, to avoid name clashes between units of different physical quantities which share the same names (like pounds). SI base units are defined as "categories" in the `unit` namespace.

Units are defined in terms of 
 1. A scale factor relative to a base unit type.
 2. A base unit
 3. [optionally] a scale factor of `pi`
 4. [optionally] a datum translation (such as the +/- 32 required to convert between `fahrenheit` and `celsius`)

 All units have their origin in the Scientific International (SI) base unit system. A special exception is made for angle units, which are defined in SI as ( m * m^-1), which is not _exactly_ the same as dimensionless/scalar units for practical purposes (and probably why the SI didn't define them as simple scalar units), and so in this library they are treated as a basic unit type.
 
_Example_: the definitions of some common length units are:

	namespace length
	{
		using meters = units::unit<std::ratio<1>, units::category::length_unit>;	// meters are (1) unit of length in the SI system.
		using feet = units::unit<std::ratio<381, 1250>, meters>;					// feet are 3.28084 meters.
	}

Unit containers
---------------

Unit containers are the workhorse of the units library, and the primary classes which will be instantiated in user code. Containers are derived from the `unit_t` class, and have the form `[unitname]_t`, e.g. `meter_t` or `radian_t`. Containers are effectively doubles with associated unit type tags, and can be used wherever a double would be used to store a dimensioned quantity.

Unit containers are defined in terms of the units they represent, their underlying type, and an optional non-linear scale (think decibels or richter scale). For example, `meter_t` would be defined: 

    using meter_t = units::unit_t<units::length::meter, double, units::linear_scale>

or simply 

    using meter_t = units::unit_t<units::length::meter>

since the underlying type and scale parameters default to `double` and `linear_scale` respectively.

Units of compatible types (e.g length units) can be implicitly converted/assigned to one another. Units (with the exception of dimensionless types) cannot be implicitly converted to/from built-in types, such as `double`. 

Units are constructed from built-in types, and the `toDouble()` method (or `operator()`) can be used to retrieve a built-in type value. That said, the user should prefer to operate within the unit type-space as much as is practical, and wrappers of most `<cmath>` functions are provided to enable operating soly in the `unit_t` domain. 

The primary purpose of unit containers is to provide type safety and dimensional analysis for mathematical operations. for instance, the velocity of an object can be calculated:

    auto objectVelocity = units::meter_t(100.0) / units::second_t(2.0);

The resulting velocity type will be deduced to be `velocity::meters_per_second` with a value of 50.0. Additionally, if the return type if specified, the type system will verify that the units are compatible. For example, the following will fail to compile:

    units::velocity::meters_per_second objectVelocity = units::square_meter_t(100.0) / units::second_t(2.0); // Error: Unit types are not compatible.`

Unit containers can (and should!) be used to perform implicit conversions:

	units::time::second_t a;
	units::time::minute_t b(1.0);
	
	a = b;	// a == 60.0

Arithmetic can be performed on unit containers the same way it can for built-in types. However, unlike built-in types, the return value of unit-type arithmetic will be the proper unit to represent the resulting quantity.

	using namespace units::length;
	using namespace units::area;
	
	meter_t a_m(1.0), b_m(2.0), c_m;
	foot_t	a_ft(1.0), b_ft(2.0), c_ft;
	
	c_m = a_m + b_m;							// OK. c == 3m
	c_ft = a_m + b_m;							// OK. resulting 3m is converted to ft.
	auto result = a_m * b_ft;					// OK. result is `meter_t` (left-most unit)
	
	auto result_sm = a_m * b_m;					// OK. result_sm is `square_meter_t`.
	auto result_s = a_m / b_m;					// OK. result_s is `dimensionless_t`.
	auto result = a_m * b_ft;					// OK. result is `square_meter_t` (left-most unit)
	
	auto result = a_m * square_meter_t(1.0);	// OK. units can always be multiplied. Result is `cubed<meter_t>`.
	auto result = a_m * scalar_t(1.0); 			// OK. units can always be multiplied. Result is `meter_t`.
	
Unsupported arithmetic, or improper return types will result in compiler errors:

	c_m = a_m + 5.0;							// Error. can't add scalars to dimensioned units.
	c_m = a_m + scalar_t(5.0);					// Error. can't add scalars to dimensioned units.
	auto result = a_m + square_meter_t(1.0);	// Error. Incompatible units.
	
By providing explicit return types for unit functions, the compiler can be used to verify the accuracy of the dimensional analysis, and thus avoiding costly errors.

`<cmath>` Functions
-------------------

The `units` library include type-safe unit_t container wrappers for almost all of the `<cmath>` functions, _including_ the c++11 extensions. These functions can be found in the `units::math` namespace. The `units` library versions don't conflict with `<cmath>`, and it's possible to use both libraries in the same code. 

The overloaded functions ensure that only the proper unit types are accepted into the functions, and that the return value type matches the expected units, all without needing to result to the type-unsafe `toDouble()` member.

In _rare_ cases, the overload resolution for a given type may be ambiguous. If so, simply prepend the function with the fully-qualified `units::math` prefix, e.g.

    meter_t x(2.0);
	meter_t y(3.0);
	square_meter_t z(1.0);
	square_meter_t result;
	
	result = fma(x, y, z);												// Error: ambiguous
	double result = fma(x.toDouble(), y.toDouble(), z.toDouble());		// Warning: Unsafe!
	result = math::fma(x, y, z);										// OK.
	
Exponentials and Square Roots
-----------------------------

Many functions require units to be raised to some power. This can be accomplished using the `units::math::pow` function:

	square_meter_t m2 = units::math::pow<2>(meter_t(5.0));	// m2 == 25.0
		
The only constraint is that the exponential power (given in the template argument) must be known at compile time, so that the type system can deduce the output type. This differs from the `<cmath> pow` implementation, which takes exponent values at runtime.

Square roots are also provided with the `units::math::sqrt` function. Due to the nature of the `sqrt` operation, the units library can often provide exact conversions for square root operations, but _not in every case_. The rest of the time, the `sqrt` unit will be a _rational_approximation_ of the real value. These are guaranteed to be accurate to at least 10 decimal places.

	meter_t m = units::math::sqrt(square_meter_t(4.0));		// m == 2.0
	
Efficiency
----------

Complex, recurively-defined conversions are performed in just 5 instructions:

		year_t twoYears(2.0);
		week_t twoYearsInWeeks = twoYears;
	00007FF7BDB57FF6  xorps       xmm9,xmm9  
	00007FF7BDB57FFA  cvtsi2sd    xmm9,rax  
	00007FF7BDB57FFF  mulsd       xmm9,mmword ptr [__real@4000000000000000 (07FF7BDBB31A0h)]  
	00007FF7BDB58008  divsd       xmm9,mmword ptr [__real@401c000000000000 (07FF7BDBB33C0h)]  
	00007FF7BDB58011  movsd       mmword ptr [rbp+6Fh],xmm9  
		EXPECT_EQ(week_t(104.286), twoYearsInWeeks);
	00007FF7BDB58017  ...

In the library, the year to week conversion is defined in terms of
`years -> days -> hours -> minutes -> seconds -> minutes -> hours -> days -> weeks`
but the total conversion ratio is computed at compile-time and the math is optimized to two floating-point operations.

Unit conversions between equivalent types are optimized away completely, and generate _no machine code_.

Compile-time Unit Manipulation
------------------------------

In many cases, unit equations are used to determine derived values from a set of values which are known at compile-time. In these situations, it would be optimal to pre-compute the derived values _at compile time_, thus generating no machine code and incurring no run-time penalty.

The `unit_value_t` class is the mechanism in the units library to perform compile-time arithmetic. The `unit_value_t` class functions exactly the same way as `std::ratio`, but with an associated unit tag and the ensuing type safety.

For a simple example, let's define a right triangle whose hypotenuse is the sum of the squares of its side (a pythagorean triple)

	struct RightTriangle
	{
		using a = unit_value_t<meters, 3>;
		using b = unit_value_t<meters, 4>;
		using c = unit_value_sqrt<unit_value_add<unit_value_power<a, 2>, unit_value_power<b, 2>>>;
	};
	
The definition above is perfectly efficient, as it generates _no run-time code_ whatsoever, and still provides all the type safety of unit containers. The values of `a`, `b`, and `c` can be accessed at runtime using the static `value()` method of `unit_value_t`

	auto a = RightTriangle::a::value();	// a is `meter_t(3)`
	auto b = RightTriangle::b::value();	// b is `meter_t(4)`
	auto c = RightTriangle::c::value();	// c is `meter_t(5)`
	
The available compile-time operations are:
 - `units::unit_value_add`
 - `units::unit_value_subtract`
 - `units::unit_value_multiply`
 - `units::unit_value_divide`
 - `units::unit_value_power`
 - `units::unit_value_sqrt`
 
Conversion without unit containers
----------------------------------

The preferred method of conversion is implicitly though the use of unit containers, however unit conversion can be accomplished using `units::convert` for arithmetic types:

	double val_in = convert<feet, inches>(1.0);	// val_in == 12.0
	
For type-safe conversion, prefer implicit conversion via unit_t type containers..

Namespaces
----------

Unit tags and containers are split into separate namespaces to avoid conflicting unit names which represent different physical quantities.

Unit tag and unit_t container definitions are defined in the following namespaces:
 - units::length
 - units::mass
 - units::time
 - units::angle (plane)
 - units::current
 - units::temperature
 - units::substance (amount of, i.e. moles)
 - units::luminous_intensity
 - units::solid_angle
 - units::frequency
 - units::velocity
 - units::angular_velocity
 - units::acceleration
 - units::force
 - units::pressure
 - units::charge
 - units::energy
 - units::power
 - units::voltage
 - units::capacitance
 - units::impedance
 - units::magnetic_flux
 - units::magnetic_field_strength
 - units::inductance
 - units::luminous_flux
 - units::illuminance
 - units::radiation
 - units::torque
 - units::area
 - units::volume
 - units::density
 - units::concentration
 - units::constants (scalar and non-scalar physical constants like Avogadro's number)
 
Mathematical operations like `sin`, `log`, `floor`, etc are defined in the following namespaces:
 - units::math
 
Type traits that you can use to test unit types are defined in the following namespaces:
 - units::traits

Defining new units
------------------

The units library strives to provide built-in types for every conceivable unit, and before defining your own units you should double-check the namespaces to make sure it's not already included. That said, if you need to roll your own units, the library is extensible by design.

Defining new units is simple, as they can be recursively defined as ratio of previously-defined units in a way that mimics natural language and is highly readable:

	namespace time
	{
		using seconds = units::unit<std::ratio<1>,   units::category::time_unit>;
		using minutes = units::unit<std::ratio<60>,  seconds>;
		using hours   = units::unit<std::ratio<60>,  minutes>;
		using days    = units::unit<std::ratio<24>,  hours>;
		using weeks   = units::unit<std::ratio<7>,   days>;
		using years   = units::unit<std::ratio<365>, days>;
	}

Units are defined in the form: `using [unit] = unit<std::ratio<[number of base units per unit]>, [base unit]>;`, where:
 - the `[unit]` is what you are defining.
 - the `[base unit]` is the unit that `[unit]` will be defined in terms of, and
 - the `[number of base units per unit]` is the conversion ratio between the two, expressed as a `std::ratio` type.
 
Compound units are defined in a similar manner, with additional helper functions for polynomials:

	using acceleration = compound_unit<meters, inverse<squared<seconds>>>;		// (m / s^2)
	
The available helpers are:
 - `units::inverse<...>`     (inverts the unit, e.g. meters becomes meters^-1, or 1 / meters)
 - `units::squared<...>`     (squares the unit, e.g. meters becomes meters^2)
 - `units::cubed<...>`       (cubes the unit, e.g. meters becomes meters^3)
 - `units::square_root<...>` (takes the square root of the unit, e.g meters^2 becomes meters)
 - `units::atto<...>` through `units::exa<...>` metric prefixes
	
Unit Type Traits
----------------

The units library provides a comprehensive set of type-traits, which can be used in templated user code to enforce that the unit types have certain properties.

For example, let's say you want to write a function that validates that the square footage of an office (given in any units), meets the minimum size required by local ordinance. 

    template<typename Units>
	bool isMinimumSize(Units x)
	{
		return x >= square_feet_t(80.0);
	}
	
This function will fail to compile if `Units` is not a unit of area (since incompatible unit types are not comparable), but it will produce a series difficult-to-understand template errors. Type traits could be used to make the error message more friendly:

    template<typename Units>
	bool isMinimumSize(Units x)
	{
		static_assert(units::traits::is_area_unit<Units>::value, "Input value x must represent an area quantity.");
		return x >= square_feet_t(80.0);
	}
	
See the `units::traits` namespace for a list of all the supported traits.

Build Instructions
------------------

The library itself consists of a single header (include/units.h), and can be included into your project without being built. 

The unit tests and documentation can be built with CMake. A doxygen installation is required to generate the documentation, and a Tex install is needed if pdf documentation is desired.

To build the tests:

Windows:
 1. Ensure cmake is installed, and that the `bin` directory is in your %PATH% variable, and that a compiler like `Visual Studio 2015 Community Edition` is installed.
 2. clone the repository or download the `.zip` package.
 3. Open a `cmd` terminal and navigate to the source directory.
 4. Type the following commands:
   - `md build`
   - `cd build`
   - `cmake -Wno-dev ..`
   - `cmake --build . --config Release`
 5. The tests will be created in an executable called `unitLibTest.exe` in the folder `build/unitTests/Release`.
   
Linux:
 1. Ensure you are using cmake 3.2 or later. You can verify this with `cmake --version`.
 2. Ensure you are using gcc version 4.9 or greater. You can verify this with `gcc --version`.
 3. clone the repository or download the `.tar.gz` package.
 4. Open a terminal and navigate to the source directory.
 5. Type the following commands:
   - `mkdir build`
   - `cd build`
   - `cmake -Wno-dev ..`
   - `cmake --build . --config Release`
 6. The tests will be created in an executable called `unitLibTest` in the folder `build/unitTests`.

Previous Releases
--------

 - v1.3.0 - Adds ostream support. bug fixes.  Tested with gcc-4.9.2, msvc2013, msvc2015.
 - v1.2.2 - Bug fixes (#1) and namespace cleanup. Tested with msvc2015, gcc 5.2.1
 - v1.2.0 - Adds angular velocity units. Tested with gcc-4.9.2, msvc2013, msvc2015.
 - v1.1.1 - Adds Doxygen and additional type traits. Tested with gcc-4.9.2, msvc2013, msvc2015.
 - v1.0.0 - Initial release. Tested with msvc2015
