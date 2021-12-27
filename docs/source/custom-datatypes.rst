Custom Datatypes
================

Requirements:

* Overload common arithmetic operators
* Define int-castable constructor (something an int can cast to)
* Arithmetic operators must be noexcept?
* Operations between int-constructed objects and objects otherwise constructed need to be noexcept


For some applications, it may be useful to define your own data type on which you would like to perform mathematical operations using xtensor. An example of this might be money. Money is fundamentally representable by a value, and so you might expect to be able to perform arithmetic operations on it. However, it also has some other properties that you may want to encapsulate in a data type:

* The value of money only has meaning in the context of a currency
* Accuracy requirements that may not be satisfied by primitive data types.

In order for a custom datatype to be usable with xtensor, the following requirements must be satisfied:

* Overload arithmetic operators ``+``, ``-``, ``*``, ``/``
* Overload comparison operators ``==``, ``!=``, ``>``, ``<``, ``>=``, ``<=``
* Implementation of a constructor that accepts a primitive integer or floating point type.

Using the above example of money, a simplistic implementation may look like the following:

.. code::

   #include <boost/multiprecision/cpp_dec_float.hpp>
   #include <string>
   
   using decimal = boost::multiprecision::cpp_dec_float_50;

   class Money {
     public:
       decimal value_;
       std::string currency_;
   
       Money() = default;
       Money(double value) : value_(value), currency_("NA"){};
       Money(decimal value, std::string currency) : value_(value), currency_(currency){};
   
       friend Money operator+(const Money &lhs, const Money &rhs) {
           return Money{lhs.value_ + rhs.value_, lhs.currency_ == "NA" ? rhs.currency_ : lhs.currency_};
       }
       friend Money operator+(const double lhs, const Money &rhs) { return Money{rhs.value_ + lhs, rhs.currency_}; }
       friend Money operator+(const Money &lhs, const double rhs) { return Money{lhs.value_ + rhs, lhs.currency_}; }
       friend Money operator-(const Money &lhs, const Money &rhs) {
           return Money{lhs.value_ - rhs.value_, lhs.currency_ == "NA" ? rhs.currency_ : lhs.currency_};
       }
       friend Money operator-(const double lhs, const Money &rhs) { return Money{lhs - rhs.value_, rhs.currency_}; }
       friend Money operator-(const Money &lhs, const double rhs) { return Money{lhs.value_ - rhs, lhs.currency_}; }
       friend Money operator*(const Money &lhs, const Money &rhs) {
           return Money{lhs.value_ * rhs.value_, lhs.currency_ == "NA" ? rhs.currency_ : lhs.currency_};
       }
       friend Money operator*(const double lhs, const Money &rhs) { return Money{lhs * rhs.value_, rhs.currency_}; }
       friend Money operator*(const Money &lhs, const double rhs) { return Money{lhs.value_ * rhs, lhs.currency_}; }
       friend Money operator/(const Money &lhs, const Money &rhs) {
           return Money{lhs.value_ / rhs.value_, lhs.currency_ == "NA" ? rhs.currency_ : lhs.currency_};
       }
       friend Money operator/(const double lhs, const Money &rhs) { return Money{lhs / rhs.value_, rhs.currency_}; }
       friend Money operator/(const Money &lhs, const double rhs) { return Money{lhs.value_ / rhs, lhs.currency_}; }
       friend bool operator==(const Money &lhs, const Money &rhs) { return lhs.value_ == rhs.value_ && lhs.currency_ == rhs.currency_; }
       friend bool operator!=(const Money &lhs, const Money &rhs) { return !(lhs == rhs); }
       friend bool operator<(const Money &lhs, const Money &rhs) { return lhs.value_ < rhs.value_; }
       friend bool operator>(const Money &lhs, const Money &rhs) { return rhs < lhs; }
       friend bool operator<=(const Money &lhs, const Money &rhs) { return !(lhs > rhs); }
       friend bool operator>=(const Money &lhs, const Money &rhs) { return !(lhs < rhs); }
       friend std::ostream &operator<<(std::ostream &stream, const Money &money) {
           stream << money.value_ << " " << money.currency_;
           return stream;
       }
   };

It is important that these operators are defined for the possible operations that you think you may need to undertake. As can be seen, operator overloads have been added for the following three cases:

* Arithmetic on two ``Money`` objects
* Arithmetic on a ``Money`` object followed by a ``double``
* Arithmetic on a ``double`` followed by a ``Money`` object

.. code::

   auto MoneyDecimal = Money{0.1234567891234567890123_dec, "USD"};
   auto MoneyLarge = Money{9876543210_dec, "USD"} + MoneyDecimal;

   xt::xarray<Money> array3{
       MoneyLarge, MoneyLarge
   };
   auto xt_mult_money = xt::prod(array3)();
   auto conventional_mult_money = MoneyLarge * MoneyLarge;
   //auto xt_pow_arr_money = xt::pow(array3, 2.0);
   auto boost_pow_money = pow(static_cast<decimal>(MoneyLarge), 2.0);

   auto bool_cmp = array3 < array3;

.. note::

   This example does not deal with basic arithmetic functions, i.e. ``sqrt``, ``pow``, etc. In order to use these, you'll also need to provide proxy functions that operate on the underlying datatype.
