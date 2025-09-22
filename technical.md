# Technical deep dive

Who is this document for?: This document is meant for anyone interested in the more nitty-gritty details and underlying
workings of the code.
I assume a basic affinity with programming concepts, but will also provide hyperlinks for any esoteric terms.

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

## Solving RPN

| Token | Action                                                                        | Stack (after action) | Note                                                      |
|-------|-------------------------------------------------------------------------------|----------------------|-----------------------------------------------------------|
| 1     | Push 1                                                                        | [1]                  | Operand                                                   |
| um    | Pop 1, apply unary minus: -1; push -1                                         | [-1]                 | Unary operator                                            |
| 2     | Push 2                                                                        | [-1, 2]              | Operand                                                   |
| 3     | Push 3                                                                        | [-1, 2, 3]           | Operand                                                   |
| ^     | Pop 3 (right), pop 2 (left): 2 ^ 3 = 8; push 8                                | [-1, 8]              | Binary operator (right-associative, but stack handles it) |
| 4     | Push 4                                                                        | [-1, 8, 4]           | Operand                                                   |
| up    | Pop 4, apply unary plus: +4; push 4                                           | [-1, 8, 4]           | Unary operator (no change)                                |
| *     | Pop 4 (right), pop 8 (left): 8 * 4 = 32; push 32                              | [-1, 32]             | Binary operator                                           |
| +     | Pop 32 (right), pop -1 (left): -1 + 32 = 31; push 31                          | [31]                 | Binary operator                                           |
| 5     | Push 5                                                                        | [31, 5]              | Operand                                                   |
| 6     | Push 6                                                                        | [31, 5, 6]           | Operand                                                   |
| um    | Pop 6, apply unary minus: -6; push -6                                         | [31, 5, -6]          | Unary operator                                            |
| /     | Pop -6 (right), pop 5 (left): 5 / -6 = -5/6; push -5/6                        | [31, -5/6]           | Binary operator                                           |
| -     | Pop -5/6 (right), pop 31 (left): 31 - (-5/6) = 31 + 5/6 = 187/6; push 187/6   | [187/6]              | Binary operator                                           |
| 7     | Push 7                                                                        | [187/6, 7]           | Operand                                                   |
| +     | Pop 7 (right), pop 187/6 (left): 187/6 + 7 = 187/6 + 42/6 = 229/6; push 229/6 | [229/6]              | Binary operator                                           |
| (end) | Stack top is the result                                                       | [229/6]              | Final result: 229/6 (or ≈ 38.1667)                        |