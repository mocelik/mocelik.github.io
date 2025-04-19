---
categories: [cpp]
date: '2025-02-26T23:54:25-05:00'
layout: post
permalink: /c/reducing-indentation-and-returning-early/
title: Reducing Indentation and Returning Early
---

Consider the following function that calculates a price based on a provided price, tax, and coupon code:

```cpp
enum class ErrorCode { INTERNAL_ERROR, BAD_COUPON };
std::expected<int, ErrorCode> calculate_total(int price, float tax, std::string coupon_code) {
    if (is_valid(coupon_code)) {
        int price_after_discounts = price - calculate_discount(coupon_code);
        if (price_after_discounts >= 0) {
            if (tax >= 1) {
                return price_after_discounts * tax;
            } else {
                return std::unexpected{ErrorCode::INTERNAL_ERROR};
            }
        } else {
            return 0;
        }
    } else {
        return std::unexpected{ErrorCode::BAD_COUPON};
    }
}
```

There are 3 nested if-statements, each with a corresponding `else` case. It’s fairly obvious that each condition statement is checking for a special case that won’t impact the general case. As we read through the function we mentally “queue” the special cases that may be handled in an `else` block down the road. Finally, when we get to the beginning of the chain of `else` blocks, we spend extra effort matching the `else` to the corresponding `if`, and again to recall the semantics of that `if`. This is a nasty context switch, even if only for a few seconds.

Consider the refactored version:

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

Now, there is a maximum of one indentation level. More importantly, we immediately see how the various cases are handled. If we’re debugging and trying to find out what happens with a bad coupon, we can stop reading this function almost immediately, whereas previously we had to manually find where the `if` block ended. Yes, most IDEs nowadays can fold the `if` expression, but that’s still an extra step to do, and more code to read.

Furthermore, the early return immediately allows readers to know that the function ends. The alternative not only involves looking for the matching `else`, they also have to keep in mind that the function may have more logic after the end of the block.

I don’t accept counter-arguments about forcing functions to do a single return. That is an antiquated practice with no place in modern Constructor Acquires, Destructor Releases (CADRe, also known as RAII) based programming. I have a hard time discerning legitimate defenders of single-return-practitioners from bad-faith arguers.
