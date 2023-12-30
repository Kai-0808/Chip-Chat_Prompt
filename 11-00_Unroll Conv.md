## 11-00_Unroll Conv
### User:
Act as a professional SystemVerilog programmer.
Unroll the for-loops of the following code. You can make shorten the code as long as the logic is the same, e.g., change if-else to `:?` operator, and change always block to assign statement.

Due to the limited text space, you can give some example of these code.

```systemverilog
always_comb begin
    temp_result = 0;

    for (i = -3; i <= 3; i++) begin
        for (j = -3; j <= 3; j++) begin
            // Zero padding for the input image
            if (RowAddr + i < 0 || RowAddr + i > 31 || ColAddr + j < 0 || ColAddr + j > 31) begin
                temp_pixel[i+3][j+3] = 0;
            end else begin
                temp_pixel[i+3][j+3] = FeatureMap[RowAddr + i][ColAddr + j];
            end

            // Assign weights based on KernelSize
            if (KernelSize == SIZE_1x1) begin
                temp_weight[i+3][j+3] = (i == 0 && j == 0) ? Weight[0][0] : 0;
            end else if (KernelSize == SIZE_3x3) begin
                temp_weight[i+3][j+3] = (i >= -1 && i <= 1 && j >= -1 && j <= 1) ? Weight[i+1][j+1] : 0;
            end else if (KernelSize == SIZE_5x5) begin
                temp_weight[i+3][j+3] = (i >= -2 && i <= 2 && j >= -2 && j <= 2) ? Weight[i+2][j+2] : 0;
            end else if (KernelSize == SIZE_7x7) begin
                temp_weight[i+3][j+3] = Weight[i+3][j+3];
            end else begin
                temp_weight[i+3][j+3] = 0;
            end

            temp_result = temp_result + temp_pixel[i+3][j+3] * temp_weight[i+3][j+3];
        end
    end
end
```

### ChatGPT:
To unroll the for-loops in the given SystemVerilog code, we will need to manually write out each iteration of the loop. Given that there are two nested loops, both of which run 7 times (from -3 to 3, inclusive), there will be a total of \(7 \times 7 = 49\) iterations to write out.

Given the space constraints, I'll break down and unroll a few iterations of the loops to demonstrate the process. Here's how to start the unrolling:

```systemverilog
always_comb begin
    temp_result = 0;

    // i = -3, j = -3
    temp_pixel[-3+3][-3+3] = (RowAddr - 3 < 0 || RowAddr - 3 > 31 || ColAddr - 3 < 0 || ColAddr - 3 > 31) ? 0 : FeatureMap[RowAddr - 3][ColAddr - 3];
    temp_weight[-3+3][-3+3] = (KernelSize == SIZE_1x1) ? ((-3 == 0 && -3 == 0) ? Weight[0][0] : 0) :
                              (KernelSize == SIZE_3x3) ? ((-3 >= -1 && -3 <= 1 && -3 >= -1 && -3 <= 1) ? Weight[-3+1][-3+1] : 0) :
                              (KernelSize == SIZE_5x5) ? ((-3 >= -2 && -3 <= 2 && -3 >= -2 && -3 <= 2) ? Weight[-3+2][-3+2] : 0) :
                              (KernelSize == SIZE_7x7) ? Weight[-3+3][-3+3] : 0;
    temp_result = temp_result + temp_pixel[-3+3][-3+3] * temp_weight[-3+3][-3+3];

    // i = -3, j = -2
    // ...
    // i = -3, j = 3
    // ...
    // i = -2, j = -3
    // ...

    // Continue in this pattern until i = 3, j = 3
end
```

This is just the beginning of the unrolled code. You would continue in this pattern for all 49 iterations, incrementing \(i\) and \(j\) as appropriate. Each iteration is similar; just update the values of \(i\) and \(j\).

Unrolling loops like this can significantly expand the codebase, making it less maintainable and harder to read. However, in hardware design, loop unrolling can lead to parallel processing, potentially speeding up the overall computation.

### User:
Give a c++ code that can generate these sv code in a file, `unrolled.sv`.

### ChatGPT:
Sure! Here's a C++ program that will generate the unrolled SystemVerilog code and save it to a file named `unrolled.sv`:

```cpp
#include <iostream>
#include <fstream>

int main() {
    std::ofstream outFile("unrolled.sv");

    if (!outFile.is_open()) {
        std::cerr << "Failed to open file for writing." << std::endl;
        return 1;
    }

    outFile << "always_comb begin\n";
    outFile << "    temp_result = 0;\n\n";

    for (int i = -3; i <= 3; i++) {
        for (int j = -3; j <= 3; j++) {
            outFile << "    // i = " << i << ", j = " << j << "\n";
            outFile << "    temp_pixel[" << i+3 << "][" << j+3 << "] = (RowAddr " << i << " < 0 || RowAddr " << i << " > 31 || ColAddr " << j << " < 0 || ColAddr " << j << " > 31) ? 0 : FeatureMap[RowAddr " << i << "][ColAddr " << j << "];\n";
            outFile << "    temp_weight[" << i+3 << "][" << j+3 << "] = (KernelSize == SIZE_1x1) ? ((" << i << " == 0 && " << j << " == 0) ? Weight[0][0] : 0) :\n";
            outFile << "                              (KernelSize == SIZE_3x3) ? ((" << i << " >= -1 && " << i << " <= 1 && " << j << " >= -1 && " << j << " <= 1) ? Weight[" << i+1 << "][" << j+1 << "] : 0) :\n";
            outFile << "                              (KernelSize == SIZE_5x5) ? ((" << i << " >= -2 && " << i << " <= 2 && " << j << " >= -2 && " << j << " <= 2) ? Weight[" << i+2 << "][" << j+2 << "] : 0) :\n";
            outFile << "                              (KernelSize == SIZE_7x7) ? Weight[" << i+3 << "][" << j+3 << "] : 0;\n";
            outFile << "    temp_result = temp_result + temp_pixel[" << i+3 << "][" << j+3 << "] * temp_weight[" << i+3 << "][" << j+3 << "];\n\n";
        }
    }

    outFile << "end";

    outFile.close();

    std::cout << "Unrolled SystemVerilog code written to unrolled.sv" << std::endl;
    return 0;
}
```

To generate the `unrolled.sv` file, simply compile and run this C++ program. You can use a compiler like `g++`:

```
g++ generator.cpp -o generator
./generator
```

This will create the `unrolled.sv` file in the same directory.

### User:
If i or j is non-negative, it will produce confused part like `ColAddr 0 < 0`. Please fix it.

### ChatGPT:
Certainly, I'll modify the C++ code to handle the generation of the `i` and `j` offsets in the SystemVerilog code more gracefully, avoiding confusing parts like `ColAddr 0 < 0`.

Here's the updated code:

```cpp
#include <iostream>
#include <fstream>

int main() {
    std::ofstream outFile("unrolled.sv");

    if (!outFile.is_open()) {
        std::cerr << "Failed to open file for writing." << std::endl;
        return 1;
    }

    outFile << "always_comb begin\n";
    outFile << "    temp_result = 0;\n\n";

    for (int i = -3; i <= 3; i++) {
        for (int j = -3; j <= 3; j++) {
            outFile << "    // i = " << i << ", j = " << j << "\n";
            outFile << "    temp_pixel[" << i+3 << "][" << j+3 << "] = (RowAddr " << (i < 0 ? i : "+ " + std::to_string(i)) << " < 0 || RowAddr " << (i < 0 ? i : "+ " + std::to_string(i)) << " > 31 || ColAddr " << (j < 0 ? j : "+ " + std::to_string(j)) << " < 0 || ColAddr " << (j < 0 ? j : "+ " + std::to_string(j)) << " > 31) ? 0 : FeatureMap[RowAddr " << (i < 0 ? i : "+ " + std::to_string(i)) << "][ColAddr " << (j < 0 ? j : "+ " + std::to_string(j)) << "];\n";
            outFile << "    temp_weight[" << i+3 << "][" << j+3 << "] = (KernelSize == SIZE_1x1) ? ((" << i << " == 0 && " << j << " == 0) ? Weight[0][0] : 0) :\n";
            outFile << "                              (KernelSize == SIZE_3x3) ? ((" << i << " >= -1 && " << i << " <= 1 && " << j << " >= -1 && " << j << " <= 1) ? Weight[" << i+1 << "][" << j+1 << "] : 0) :\n";
            outFile << "                              (KernelSize == SIZE_5x5) ? ((" << i << " >= -2 && " << i << " <= 2 && " << j << " >= -2 && " << j << " <= 2) ? Weight[" << i+2 << "][" << j+2 << "] : 0) :\n";
            outFile << "                              (KernelSize == SIZE_7x7) ? Weight[" << i+3 << "][" << j+3 << "] : 0;\n";
            outFile << "    temp_result = temp_result + temp_pixel[" << i+3 << "][" << j+3 << "] * temp_weight[" << i+3 << "][" << j+3 << "];\n\n";
        }
    }

    outFile << "end";

    outFile.close();

    std::cout << "Unrolled SystemVerilog code written to unrolled.sv" << std::endl;
    return 0;
}
```

Now, when `i` or `j` is non-negative, the code will produce statements like `ColAddr + 0 < 0`, which should be clearer.

### User:
.\unroll.cpp: In function 'int main()':
.\unroll.cpp:18:93: error: operands to ?: have different types 'int' and 'std::__cxx11::basic_string<char>'  
             outFile << "    temp_pixel[" << i+3 << "][" << j+3 << "] = (RowAddr " << (i < 0 ? i : "+ " + std::to_string(i)) << " < 0 || RowAddr " << (i < 0 ? i : "+ " + std::to_string(i)) << " > 31 || ColAddr " << (j < 0 ? j : "+ " + std::to_string(j)) << " < 0 || ColAddr " << (j < 0 ? j : "+ " + std::to_string(j)) << " > 31) ? 0 : FeatureMap[RowAddr " << (i < 0 ? i : "+ " + std::to_string(i)) << "][ColAddr " << (j < 0 ? j : "+ " + std::to_string(j)) << "];\n";
                                                                                       ~~~~~~^~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
.\unroll.cpp:18:157: error: operands to ?: have different types 'int' and 'std::__cxx11::basic_string<char>' 
             outFile << "    temp_pixel[" << i+3 << "][" << j+3 << "] = (RowAddr " << (i < 0 ? i : "+ " + std::to_string(i)) << " < 0 || RowAddr " << (i < 0 ? i : "+ " + std::to_string(i)) << " > 31 || ColAddr " << (j < 0 ? j : "+ " + std::to_string(j)) << " < 0 || ColAddr " << (j < 0 ? j : "+ " + std::to_string(j)) << " > 31) ? 0 : FeatureMap[RowAddr " << (i < 0 ? i : "+ " + std::to_string(i)) << "][ColAddr " << (j < 0 ? j : "+ " + std::to_string(j)) << "];\n";
                                                                                                             
                                          ~~~~~~^~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
.\unroll.cpp:18:222: error: operands to ?: have different types 'int' and 'std::__cxx11::basic_string<char>' 
             outFile << "    temp_pixel[" << i+3 << "][" << j+3 << "] = (RowAddr " << (i < 0 ? i : "+ " + std::to_string(i)) << " < 0 || RowAddr " << (i < 0 ? i : "+ " + std::to_string(i)) << " > 31 || ColAddr " << (j < 0 ? j : "+ " + std::to_string(j)) << " < 0 || ColAddr " << (j < 0 ? j : "+ " + std::to_string(j)) << " > 31) ? 0 : FeatureMap[RowAddr " << (i < 0 ? i : "+ " + std::to_string(i)) << "][ColAddr " << (j < 0 ? j : "+ " + std::to_string(j)) << "];\n";
                                                                                                             
                                                                                                           ~~~~~~^~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
.\unroll.cpp:18:286: error: operands to ?: have different types 'int' and 'std::__cxx11::basic_string<char>' 
             outFile << "    temp_pixel[" << i+3 << "][" << j+3 << "] = (RowAddr " << (i < 0 ? i : "+ " + std::to_string(i)) << " < 0 || RowAddr " << (i < 0 ? i : "+ " + std::to_string(i)) << " > 31 || ColAddr " << (j < 0 ? j : "+ " + std::to_string(j)) << " < 0 || ColAddr " << (j < 0 ? j : "+ " + std::to_string(j)) << " > 31) ? 0 : FeatureMap[RowAddr " << (i < 0 ? i : "+ " + std::to_string(i)) << "][ColAddr " << (j < 0 ? j : "+ " + std::to_string(j)) << "];\n";
                                                                                                             
                                                                                                             
                                                              ~~~~~~^~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
.\unroll.cpp:18:366: error: operands to ?: have different types 'int' and 'std::__cxx11::basic_string<char>' 
             outFile << "    temp_pixel[" << i+3 << "][" << j+3 << "] = (RowAddr " << (i < 0 ? i : "+ " + std::to_string(i)) << " < 0 || RowAddr " << (i < 0 ? i : "+ " + std::to_string(i)) << " > 31 || ColAddr " << (j < 0 ? j : "+ " + std::to_string(j)) << " < 0 || ColAddr " << (j < 0 ? j : "+ " + std::to_string(j)) << " > 31) ? 0 : FeatureMap[RowAddr " << (i < 0 ? i : "+ " + std::to_string(i)) << "][ColAddr " << (j < 0 ? j : "+ " + std::to_string(j)) << "];\n";
                                                                                                             
                                                                                                             
                                                                                                             
                                 ~~~~~~^~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
.\unroll.cpp:18:424: error: operands to ?: have different types 'int' and 'std::__cxx11::basic_string<char>' 
             outFile << "    temp_pixel[" << i+3 << "][" << j+3 << "] = (RowAddr " << (i < 0 ? i : "+ " + std::to_string(i)) << " < 0 || RowAddr " << (i < 0 ? i : "+ " + std::to_string(i)) << " > 31 || ColAddr " << (j < 0 ? j : "+ " + std::to_string(j)) << " < 0 || ColAddr " << (j < 0 ? j : "+ " + std::to_string(j)) << " > 31) ? 0 : FeatureMap[RowAddr " << (i < 0 ? i : "+ " + std::to_string(i)) << "][ColAddr " << (j < 0 ? j : "+ " + std::to_string(j)) << "];\n";
                                                                                                             
                                                                                                             
                                                                                                             
                                                                                           ~~~~~~^~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

### ChatGPT:
Apologies for the oversight. The ternary operator expects both outcomes to be of the same type. In the code I provided, we have one outcome as an `int` and the other as a `std::string`.

Here's a revised version of the C++ code that should compile and run correctly:

```cpp
#include <iostream>
#include <fstream>

std::string offset(int value) {
    if (value < 0) return std::to_string(value);
    else if (value == 0) return "";
    else return "+ " + std::to_string(value);
}

int main() {
    std::ofstream outFile("unrolled.sv");

    if (!outFile.is_open()) {
        std::cerr << "Failed to open file for writing." << std::endl;
        return 1;
    }

    outFile << "always_comb begin\n";
    outFile << "    temp_result = 0;\n\n";

    for (int i = -3; i <= 3; i++) {
        for (int j = -3; j <= 3; j++) {
            outFile << "    // i = " << i << ", j = " << j << "\n";
            outFile << "    temp_pixel[" << i+3 << "][" << j+3 << "] = (RowAddr " << offset(i) << " < 0 || RowAddr " << offset(i) << " > 31 || ColAddr " << offset(j) << " < 0 || ColAddr " << offset(j) << " > 31) ? 0 : FeatureMap[RowAddr " << offset(i) << "][ColAddr " << offset(j) << "];\n";
            outFile << "    temp_weight[" << i+3 << "][" << j+3 << "] = (KernelSize == SIZE_1x1) ? ((" << i << " == 0 && " << j << " == 0) ? Weight[0][0] : 0) :\n";
            outFile << "                              (KernelSize == SIZE_3x3) ? ((" << i << " >= -1 && " << i << " <= 1 && " << j << " >= -1 && " << j << " <= 1) ? Weight[" << i+1 << "][" << j+1 << "] : 0) :\n";
            outFile << "                              (KernelSize == SIZE_5x5) ? ((" << i << " >= -2 && " << i << " <= 2 && " << j << " >= -2 && " << j << " <= 2) ? Weight[" << i+2 << "][" << j+2 << "] : 0) :\n";
            outFile << "                              (KernelSize == SIZE_7x7) ? Weight[" << i+3 << "][" << j+3 << "] : 0;\n";
            outFile << "    temp_result = temp_result + temp_pixel[" << i+3 << "][" << j+3 << "] * temp_weight[" << i+3 << "][" << j+3 << "];\n\n";
        }
    }

    outFile << "end";

    outFile.close();

    std::cout << "Unrolled SystemVerilog code written to unrolled.sv" << std::endl;
    return 0;
}
```

I introduced a helper function `offset()` to manage the creation of the offset string based on the value of `i` or `j`. This should resolve the compilation errors.

### User:
For the part like `outFile << "                              (KernelSize == SIZE_3x3) ? ((" << i << " >= -1 && " << i << " <= 1 && " << j << " >= -1 && " << j << " <= 1) ? Weight[" << i+1 << "][" << j+1 << "] : 0) :\n";`
It will print the condition `(i >= -1 && i <= 1 && j >= -1 && j <= 1) ?` out.
Compare the value first, and then print the corresponding value.

### ChatGPT:
Understood. I'll update the code to evaluate the conditions for `i` and `j` first, and then print the appropriate value based on the evaluation. Here's the revised code:

```cpp
#include <iostream>
#include <fstream>

std::string offset(int value) {
    if (value < 0) return std::to_string(value);
    else if (value == 0) return "";
    else return "+ " + std::to_string(value);
}

std::string weightValue(int i, int j) {
    if (i == 0 && j == 0) return "Weight[0][0]";
    else if (i >= -1 && i <= 1 && j >= -1 && j <= 1) return "Weight[" + std::to_string(i+1) + "][" + std::to_string(j+1) + "]";
    else if (i >= -2 && i <= 2 && j >= -2 && j <= 2) return "Weight[" + std::to_string(i+2) + "][" + std::to_string(j+2) + "]";
    else return "Weight[" + std::to_string(i+3) + "][" + std::to_string(j+3) + "]";
}

int main() {
    std::ofstream outFile("unrolled.sv");

    if (!outFile.is_open()) {
        std::cerr << "Failed to open file for writing." << std::endl;
        return 1;
    }

    outFile << "always_comb begin\n";
    outFile << "    temp_result = 0;\n\n";

    for (int i = -3; i <= 3; i++) {
        for (int j = -3; j <= 3; j++) {
            outFile << "    // i = " << i << ", j = " << j << "\n";
            outFile << "    temp_pixel[" << i+3 << "][" << j+3 << "] = (RowAddr " << offset(i) << " < 0 || RowAddr " << offset(i) << " > 31 || ColAddr " << offset(j) << " < 0 || ColAddr " << offset(j) << " > 31) ? 0 : FeatureMap[RowAddr " << offset(i) << "][ColAddr " << offset(j) << "];\n";
            outFile << "    temp_weight[" << i+3 << "][" << j+3 << "] = (KernelSize == SIZE_1x1) ? (" << weightValue(i, j) << ") :\n";
            outFile << "                              (KernelSize == SIZE_3x3) ? (" << weightValue(i, j) << ") :\n";
            outFile << "                              (KernelSize == SIZE_5x5) ? (" << weightValue(i, j) << ") :\n";
            outFile << "                              (KernelSize == SIZE_7x7) ? (" << weightValue(i, j) << ") : 0;\n";
            outFile << "    temp_result = temp_result + temp_pixel[" << i+3 << "][" << j+3 << "] * temp_weight[" << i+3 << "][" << j+3 << "];\n\n";
        }
    }

    outFile << "end";

    outFile.close();

    std::cout << "Unrolled SystemVerilog code written to unrolled.sv" << std::endl;
    return 0;
}
```

The function `weightValue(int i, int j)` will return the correct weight value based on the conditions for `i` and `j`. This way, the condition won't be printed in the output file, and only the appropriate value will be printed.

### User:
You may use four different functions for the four cases.

### ChatGPT:
Understood. I'll split the weight assignment logic into four different functions to handle each of the cases (`SIZE_1x1`, `SIZE_3x3`, `SIZE_5x5`, and `SIZE_7x7`). 

Here's the revised code:

```cpp
#include <iostream>
#include <fstream>

std::string offset(int value) {
    if (value < 0) return std::to_string(value);
    else if (value == 0) return "";
    else return "+ " + std::to_string(value);
}

std::string weightValue_1x1(int i, int j) {
    return (i == 0 && j == 0) ? "Weight[0][0]" : "0";
}

std::string weightValue_3x3(int i, int j) {
    return (i >= -1 && i <= 1 && j >= -1 && j <= 1) ? ("Weight[" + std::to_string(i+1) + "][" + std::to_string(j+1) + "]") : "0";
}

std::string weightValue_5x5(int i, int j) {
    return (i >= -2 && i <= 2 && j >= -2 && j <= 2) ? ("Weight[" + std::to_string(i+2) + "][" + std::to_string(j+2) + "]") : "0";
}

std::string weightValue_7x7(int i, int j) {
    return "Weight[" + std::to_string(i+3) + "][" + std::to_string(j+3) + "]";
}

int main() {
    std::ofstream outFile("unrolled.sv");

    if (!outFile.is_open()) {
        std::cerr << "Failed to open file for writing." << std::endl;
        return 1;
    }

    outFile << "always_comb begin\n";
    outFile << "    temp_result = 0;\n\n";

    for (int i = -3; i <= 3; i++) {
        for (int j = -3; j <= 3; j++) {
            outFile << "    // i = " << i << ", j = " << j << "\n";
            outFile << "    temp_pixel[" << i+3 << "][" << j+3 << "] = (RowAddr " << offset(i) << " < 0 || RowAddr " << offset(i) << " > 31 || ColAddr " << offset(j) << " < 0 || ColAddr " << offset(j) << " > 31) ? 0 : FeatureMap[RowAddr " << offset(i) << "][ColAddr " << offset(j) << "];\n";
            outFile << "    temp_weight[" << i+3 << "][" << j+3 << "] = (KernelSize == SIZE_1x1) ? (" << weightValue_1x1(i, j) << ") :\n";
            outFile << "                              (KernelSize == SIZE_3x3) ? (" << weightValue_3x3(i, j) << ") :\n";
            outFile << "                              (KernelSize == SIZE_5x5) ? (" << weightValue_5x5(i, j) << ") :\n";
            outFile << "                              (KernelSize == SIZE_7x7) ? (" << weightValue_7x7(i, j) << ") : 0;\n";
            outFile << "    temp_result = temp_result + temp_pixel[" << i+3 << "][" << j+3 << "] * temp_weight[" << i+3 << "][" << j+3 << "];\n\n";
        }
    }

    outFile << "end";

    outFile.close();

    std::cout << "Unrolled SystemVerilog code written to unrolled.sv" << std::endl;
    return 0;
}
```

With this setup, each kernel size (`SIZE_1x1`, `SIZE_3x3`, `SIZE_5x5`, and `SIZE_7x7`) has its own dedicated function to compute the weight value. This makes the code more modular and easy to maintain.
