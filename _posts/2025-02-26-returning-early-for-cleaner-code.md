---
categories: [cpp]
date: '2025-02-26T23:54:25-05:00'
layout: post
permalink: /c/returning-early-for-cleaner-code/
title: Return Early for Cleaner Code
---

Surprisingly, returning early from a function is a controversial topic. This post is yet another one suggesting to return early, adding to the existing literature of [blogs](https://blog.codinghorror.com/flattening-arrow-code/), [more blogs](https://gis.utah.gov/blog/2021-12-29-python-shorts-arrow-code/), [stack exchange questions](https://softwareengineering.stackexchange.com/questions/18454/should-i-return-from-a-function-early-or-use-an-if-statement), and [stack overflow questions](https://stackoverflow.com/questions/355670/is-returning-early-from-a-function-more-elegant-than-an-if-statement).

I find early returns, i.e. guard clauses, to improve code readability by reducing both nesting, context switches, and the amount of code to read when tracing code.

Consider the following function that calculates a total price based on a subtotal price, tax rate, and coupon code:

```cpp
enum class ErrorCode { INTERNAL_ERROR, BAD_COUPON };
std::expected<int, ErrorCode> calculate_total(int price, float tax, std::string coupon_code) {
    std::expected<int, ErrorCode> rc;
    if (is_valid(coupon_code)) {
        int price_after_discounts = price - calculate_discount(coupon_code);
        if (price_after_discounts >= 0) {
            if (tax >= 1) {
                rc = price_after_discounts * tax;
            } else {    // tax should not be < 1
                rc = std::unexpected{ErrorCode::INTERNAL_ERROR};
            }
        } else {    // do not discount more than the total price
            rc = 0;
        }
    } else {    // coupon is invalid
        rc = std::unexpected{ErrorCode::BAD_COUPON};
    }
    return rc;
}
```

There are 3 nested if-statements, each with a corresponding `else` case. As we read through the function, we mentally “queue” the special cases (bad coupon or invalid tax rate) while we look at the success path. Then, we finally see the meat of the function, 3 indentations deep.

If we're trying to trace a failed transaction, we have to read not only the success case but also begin unwinding our mental queue of failure cases as we begin to traverse the chain of else statements. Thankfully, the comments help guide that path, but we all know that comments age faster than code. Furthermore, there's a mental context switch as we go from `Check failure 1 -> Check failure 2 -> Check failure 3 -> Calculate success value -> Calculate failure 1 value -> Calculate failure 2 value -> Calculate failure 3 value -> Return value from one of 4 cases`.

Consider the refactored version that uses returns early using guard clauses:

```cpp
enum class ErrorCode { INTERNAL_ERROR, BAD_COUPON };
std::expected<int, ErrorCode> calculate_total(int price, float tax, std::string coupon_code) {
    if (!is_valid(coupon_code)) {
        return std::unexpected{ErrorCode::BAD_COUPON};
    }

    int price_after_discounts = price - calculate_discount(coupon_code);
    if (price_after_discounts < 0) {
        return 0;
    }

    if (tax < 1) {
        return std::unexpected{ErrorCode::INTERNAL_ERROR};
    }

    return price_after_discounts * tax;
}
```

Now, there is a maximum of one indentation level. More importantly, we immediately see how the various cases are handled. The mental stack sticks with the case: `Check failure 1 -> Return failure 1 value`, `Check failure 2 -> Return failure 2 value`, `Check failure 3 -> Return failure 3 value`, `Return success value`. After each failure case, we can safely forget that scenario as we read the rest of the function since we know we have an invariant, or that a contract has been fulfilled.

Another way of thinking about it is that if we’re debugging and trying to find out what happens with a bad coupon, we can stop reading this function almost immediately, whereas previously we had to manually find where the `if` block ended. Yes, most IDEs nowadays can fold the `if` expression for us, but that still involves manual work before we see more code.

Furthermore, the early return immediately allows readers to know that the function ends. The alternative not only involves looking for the matching `else`, they also have to keep in mind that the function may have more logic after the end of the block.

I have heard counter-arguments about forgetting to clean up resources. That does not happen in any modern Constructor Acquires, Destructor Releases (CADRe, also known as RAII) based programming. Even in languages that don't support such constructs, like C, a CADRe-based approach is the common way of handling things, with `return` replaced with a `goto` to a label at the end of the function that cleans up any allocated resources.

I have heard more counter-arguments about functional programming not supporting CADRe, specifically in the context of C++. While I don't know every language out there, I do know that C++ supports `scope_exit`, `scope_success`, and `scope_fail` to run a function whenever a scope exits, exits normally (via a return statement), or exits via a thrown exception respectively. While these are not yet officially parts of the C++ standard (as of early 2025), they are supported by most Standard libraries, and might even be easy enough to implement manually. Destructors and CADRe can be used with functional programming.

Please, encourage returning early.
