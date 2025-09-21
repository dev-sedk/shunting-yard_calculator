# Technical deep dive

Who is this document for?: This document is meant for anyone interested in the more nitty gritty details and underlying
workings of the code.
I assume a basic level of programming knowledge, but will also provide hyperlinks for esoteric terms.

What does document do?: It outlines the more technical aspects of this project, like the algorithms used and code
conventions.

## The Shunting Yard algorithm

This algorithm results in an [RPN](https://en.wikipedia.org/wiki/Reverse_Polish_notation) representation of
the [infix notation](https://en.wikipedia.org/wiki/Infix_notation) it takes in.
Following are the steps the algorithm would take to transform `1 + 2 * 3 * (-5 + 2)` into RPN.

| Token     | Action                                                                                                                                             | Output                             | Stack    | Note                                                                              |
|-----------|----------------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------|----------|-----------------------------------------------------------------------------------|
| - (unary) | Append to stack (unary minus, highest precedence)                                                                                                  |                                    | um       | Unary minus (represented as um)                                                   |
| 1         | Append to output                                                                                                                                   | 1                                  | um       | Number                                                                            |
| +         | Pop um (higher precedence) to output; append + to stack                                                                                            | 1 um                               | +        | Binary operator                                                                   |
| 2         | Append to output                                                                                                                                   | 1 um 2                             | +        | Number                                                                            |
| ^         | Append to stack (higher precedence than +)                                                                                                         | 1 um 2                             | + ^      | Binary operator (right-associative)                                               |
| 3         | Append to output                                                                                                                                   | 1 um 2 3                           | + ^      | Number                                                                            |
| *         | Pop ^ (higher precedence) to output; append * to stack                                                                                             | 1 um 2 3 ^                         | + *      | Binary operator                                                                   |
| + (unary) | Append to stack (unary plus, highest precedence)                                                                                                   | 1 um 2 3 ^                         | + * up   | Unary plus (represented as up)                                                    |
| 4         | Append to output                                                                                                                                   | 1 um 2 3 ^ 4                       | + * up   | Number                                                                            |
| -         | Pop up (higher precedence) to output; pop * (higher precedence) to output; pop + (equal precedence, left-associative) to output; append - to stack | 1 um 2 3 ^ 4 up * +                | -        | Binary operator                                                                   |
| (         | Append to stack                                                                                                                                    | 1 um 2 3 ^ 4 up * +                | - (      | Left parenthesis                                                                  |
| 5         | Append to output                                                                                                                                   | 1 um 2 3 ^ 4 up * + 5              | - (      | Number                                                                            |
| /         | Append to stack                                                                                                                                    | 1 um 2 3 ^ 4 up * + 5              | - ( /    | Binary operator                                                                   |
| - (unary) | Append to stack (unary minus, highest precedence)                                                                                                  | 1 um 2 3 ^ 4 up * + 5              | - ( / um | Unary minus (represented as um)                                                   |
| 6         | Append to output                                                                                                                                   | 1 um 2 3 ^ 4 up * + 5 6            | - ( / um | Number                                                                            |
| )         | Pop operators until (: pop um to output, pop / to output, pop (                                                                                    | 1 um 2 3 ^ 4 up * + 5 6 um /       | -        | Right parenthesis                                                                 |
| +         | Pop - (equal precedence, left-associative) to output; append + to stack                                                                            | 1 um 2 3 ^ 4 up * + 5 6 um / -     | +        | Binary operator                                                                   |
| 7         | Append to output                                                                                                                                   | 1 um 2 3 ^ 4 up * + 5 6 um / - 7   | +        | Number                                                                            |
| (end)     | Pop remaining operators to output: pop +                                                                                                           | 1 um 2 3 ^ 4 up * + 5 6 um / - 7 + |          | Final RPN: 1 um 2 3 ^ 4 up * + 5 6 um / - 7 + (um = unary minus, up = unary plus) |
