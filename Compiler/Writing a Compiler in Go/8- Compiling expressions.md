### Cleaning up the stack
At this point the only thing my compiler and my VM are capable of is adding two numbers. Given 1 + 2, the VM will correctly put 3 on its stack. That's good, but the problem is that the 3 stayus on the stack for all eternity. 

As a quick refresh, I have three types of statements in Monkey: let statements, return statements and expression statements. The problem is not with the expression 1 + 2 itself, but rather where is occurs. It's part of an expression statement. The first two explicitly reuse the values their child-expression nodes produce, expression statements merely wrap expressions so they can occur on their own. The value they produces is not reused, by definition. But now the problem is that I do reuse it, because I involuntarily keep it on the stack. 
```Monkey
1;
2;
3;
```

That's three separate expression statements. And what ends up on the stack? Not just the value produced last, 3, but everything: 1, 2 and 3. If I have a Monkey program consisting of lots of expression statements I could fill up the stack by accident.
To fix this, I need to do two things: First, define a new opcode that tells the VM to pop the topmost element of the stack. Second, emit this opcode after every expression statement.

Let's define the opcode, **OpPop**
```go
// code/code.go
const (
	OpConstant Opcode = iota
	OpAdd
	OpPop
)

var definitions = map[Opcode]*Definition{
	OpConstant: {"OpConstant", []int{2}},
	OpAdd:      {"OpAdd", []int{}},
	OpPop:      {"OpPop", []int{}},
}
```

Just like **OpAdd**, **OpPop** does not have operands. Its only job is to tell the VM to pop the topmost element off the stack and for that it doesn't need an operand.
Now I need to use this opcode to clean the stack after every expression statement. To this, let's start by tests, and change **TestIntegerArithmetic**:
```go
// compiler/compiler_test.go
func TestIntegerArithmetic(t *testing.T) {
	tests := []compilerTestCase{
		{
			input:             "1 + 2",
			expectedConstants: []interface{}{1, 2},
			expectedInstructions: []code.Instructions{
				code.Make(code.OpConstant, 0),
				code.Make(code.OpConstant, 1),
				code.Make(code.OpAdd),
				code.Make(code.OpPop),
			},
		},
	}

	runCompilerTests(t, tests)
}
```

The only change here is the new line containing the **code.Make(code.OpPop)** call. I assert that the compiled expression statement should be followed by an **OpPop** instruction. The desired behavior can be made even clearer by adding another test with multiple expression statements:
```go
// compiler/compiler_test.go
func TestIntegerArithmetic(t *testing.T) {
	tests := []compilerTestCase{
		{
			input:             "1 + 2",
			expectedConstants: []interface{}{1, 2},
			expectedInstructions: []code.Instructions{
				code.Make(code.OpConstant, 0),
				code.Make(code.OpConstant, 1),
				code.Make(code.OpAdd),
				code.Make(code.OpPop),
			},
		},
		{
			input:             "1; 2",
			expectedConstants: []interface{}{1, 2},
			expectedInstructions: []code.Instructions{
				code.Make(code.OpConstant, 0),
				code.Make(code.OpPop),
				code.Make(code.OpConstant, 1),
				code.Make(code.OpPop),
			},
		},
	}

	runCompilerTests(t, tests)
}
```

Note that the **;** that separates the **1** from **2**. Both integers literals are separate expression statements and after each statement an **OpPop** instruction should be emitted. And, ofc, that's not what currently happens. Instead I tell my VM to fill up its stack by loading constants on to it:
```shell
go test ./compiler 
--- FAIL: TestIntegerArithmetic (0.00s)
    compiler_test.go:112: testInstructions failed: _wrong instructions length.
        want="0000 OpConstant 0\n0003 OpConstant 1\n0006 OpAdd\n0007 OpPop\n"
        got="0000 OpConstant 0\n0003 OpConstant 1\n0006 OpAdd\n"
FAIL
FAIL    github.com/vit0rr/mumu/compiler 0.526s
FAIL
```

But - that's the good part, in order to fix this and properly clean the stack, all I need to do is to add a single call to **c.emit** to my compiler:
```go
// compiler/compiler.go
func (c *Compiler) Compile(node ast.Node) error {
    // [...]
	case *ast.ExpressionStatement:
		err := c.Compile(node.Expression)
		if err != nil {
			return err
		}
		c.emit(code.OpPop)
```

After compiling the **node.Expression** of an `*ast.ExpressionStatement` I emit an **OpPop**. That's all it takes:
```shell
go test ./compiler
ok      github.com/vit0rr/mumu/compiler 0.499s
```

That's not all it takes. I still need to work left to do, because now I need to tell the VM how to handle this **OpPop** instruction, which would also be a tiny addition of it weren't for my tests.

In VM tests I used **vm.StackTop** to make sure that VM put the correct things on to its stack, but with **OpPop** in play I can't do that anymore. Now, what I wanna assert is that "this should have been on the stack, right before you popped it off, dear VM". In order to do that, I add a test-only method to my VM and get rid of **StackTop**:
```go
// vm/vm.go
func (vm *VM) LastPoppedStackElem() object.Object {
	return vm.stack[vm.sp]
}
```

**vm.sp** always point to the next free slot in **vm.stack**. This is where a new element would be pushed. But since I only pop elements off the stack by decrementing **vm.sp** (without explicitly setting them to nil), this is also where I can find the elements that were previously on top of the stack. With **LastPoppedStackElem**, I can make change VM tests to make sure **OpPop** is actually handled correctly:
```go
// vm/vm_test.go
func runVmTests(t *testing.T, tests []vmTestCase) {
	t.Helper()

	// [...]

		stackElem := vm.LastPoppedStackElem()

		testExpectedObject(t, tt.expected, stackElem)
	}
}
```
And this is enough to break mu tests:
```shell
go test ./vm      
--- FAIL: TestIntegerArithmetic (0.00s)
    vm_test.go:86: testIntegerObject failed: object is not Integer. got=<nil> (<nil>)
    vm_test.go:86: testIntegerObject failed: object is not Integer. got=<nil> (<nil>)
    vm_test.go:86: testIntegerObject failed: object has wrong value. got=2, want=3
FAIL
FAIL    github.com/vit0rr/mumu/vm       0.799s
FAIL
```

In order to get them to pass again, I need to tell the VM to keep its stack clean and tidy:
```go
// vm/vm.go
func (vm *VM) Run() error {
    // [...]
		case code.OpPop:
			vm.pop()
		}
	}

	return nil
}
```
With that, stack hygiene is restored:
```shell
go test ./vm  
ok      github.com/vit0rr/mumu/vm       0.460s
```

And I also need to fix my REPL, where I still use **StackTop**, by replacing it with **LastPoppedStackElem**:
```go
// repl/repl.go
func Start(in io.Reader, out io.Writer, useCompiler bool) {
	// [...]
			lastPopped := machine.LastPoppedStackElem()
			io.WriteString(out, lastPopped.Inspect())
			io.WriteString(out, "\n")

			continue
		}
```

And on main:
```go
// main.go
	if useCompiler {
		// [...]

		lastPopped := machine.LastPoppedStackElem()
		io.WriteString(os.Stdout, lastPopped.Inspect())
		io.WriteString(os.Stdout, "\n")

		return nil
	}
```

That means I can move on and safely do more arithmetic on the stack without the stack slowly blowing up in my face.

### Infix Expressions
To remember, Monkey supports eight (8) infix operators and four (4) of them are being used for arithmetic: **+**, **-**, `*`, and **/**. I've already added support for **+** with the **OpAdd** opcode. Now I need to add three more. And since all three of them work the same way in regards to their use of operands and the stack, I'll add them together.

Let's add the **Opcode** definitions to the **code** package:
```go
// code/code.go
const (
	OpConstant Opcode = iota
	OpPop
	OpAdd
	OpSub
	OpMul
	OpDiv
)

var definitions = map[Opcode]*Definition{
	OpConstant: {"OpConstant", []int{2}},
	OpPop:      {"OpPop", []int{}},
	OpAdd:      {"OpAdd", []int{}},
	OpSub:      {"OpSub", []int{}},
	OpMul:      {"OpMul", []int{}},
	OpDiv:      {"OpDiv", []int{}},
}
```

It's kind of obvious, but **OpSub** stands for the **-**, **OpMul** for the `*`, and **OpDiv** for the **/** infix operator. With these opcodes defined, I can use them in my compiler tests to make sure the compiler knows how to output them:
```go
// compiler/compiler_test.go
func TestIntegerArithmetic(t *testing.T) {
        // [...]
		{
			input:             "1 - 2",
			expectedConstants: []interface{}{1, 2},
			expectedInstructions: []code.Instructions{
				code.Make(code.OpConstant, 0),
				code.Make(code.OpConstant, 1),
				code.Make(code.OpSub),
				code.Make(code.OpPop),
			},
		},
		{
			input:             "1 * 2",
			expectedConstants: []interface{}{1, 2},
			expectedInstructions: []code.Instructions{
				code.Make(code.OpConstant, 0),
				code.Make(code.OpConstant, 1),
				code.Make(code.OpMul),
				code.Make(code.OpPop),
			},
		},
		{
			input:             "2 / 1",
			expectedConstants: []interface{}{2, 1},
			expectedInstructions: []code.Instructions{
				code.Make(code.OpConstant, 0),
				code.Make(code.OpConstant, 1),
				code.Make(code.OpDiv),
				code.Make(code.OpPop),
			},
		},
	}

	runCompilerTests(t, tests)
}
```

The only thing different here is the last test case, where I changed the order of the operands. The rest, are pretty similar. The test ofc not pass. I need to teach the compiler how to read these infix expression on switch statement:

```go
// compiler/compiler.go
func (c *Compiler) Compile(node ast.Node) error {
    // [...] 

	case *ast.InfixExpression:
		err := c.Compile(node.Left)
		if err != nil {
			return err
		}

		err = c.Compile(node.Right)
		if err != nil {
			return err
		}

		switch node.Operator {
		case "+":
			c.emit(code.OpAdd)
		case "-":
			c.emit(code.OpSub)
		case "*":
			c.emit(code.OpMul)
		case "/": 
			c.emit(code.OpDiv)
		default:
			return fmt.Errorf("unknow operador %s", node.Operator)
		}

	case *ast.IntegerLiteral:
		integer := &object.Integer{Value: node.Value}
		c.emit(code.OpConstant, c.addConstant(integer))
	}

	return nil
}
```

And they make the tests pass:
```shell
go test ./compiler
ok      github.com/vit0rr/mumu/compiler 0.471s
```

It means that now the compiler outputs three more opcodes. The VM must now step up to this challenge. 
```go
// vm/vm_test.go
func TestIntegerArithmetic(t *testing.T) {
	tests := []vmTestCase{
		{"1", 1},
		{"2", 2},
		{"1 + 2", 3},
		{"1 - 2", -1},
		{"1 * 2", 2},
		{"4 / 2", 2},
		{"50 / 2 * 2 + 10 - 2", 55},
		{"5 + 5 + 5 + 5 - 10", 10},
		{"2 * 2 * 2 * 2 * 2", 32},
		{"5 * 2 + 10", 20},
		{"5 + 2 * 10", 25},
		{"5 * (2 + 10)", 60},
	}

	runVmTests(t, tests)
}
```

I not only have three test cases necessary to make sure the **OpSub, OpMul** and **OpDiv** opcodes are recognized by the VM, but there's also a series of tests cases that mix the infix operators, playing with their varying levels of precedence and manipulating them by hand with added parentheses. They all fail for now. But the required changes to make all of them pass are minimal:
```go
// vm/vm.go
func (vm *VM) Run() error {
    // [...]

		case code.OpAdd, code.OpSub, code.OpMul, code.OpDiv:
			err := vm.executeBinaryOperation(op)
			if err != nil {
				return err
			}

		case code.OpPop:
			vm.pop()
		}
	}

	return nil
}
```

```go
// vm/vm.go
func (vm *VM) executeBinaryOperation(op code.Opcode) error {
	right := vm.pop()
	left := vm.pop()

	leftType := left.Type()
	rightType := right.Type()

	if leftType == object.INTEGER_OBJ && rightType == object.INTEGER_OBJ {
		return vm.executeBinaryIntegerOperation(op, left, right)
	}

	return fmt.Errorf("unsupported types for binary operation: %s %s", leftType, rightType)
}
```

It doesn't do much more than type assertions and possily producing an error and delegates most of the work to **executeBinaryIntegerOperation**:

```go
func (vm *VM) executeBinaryIntegerOperation(
	op code.Opcode,
	left, right object.Object,
) error {
	leftValue := left.(*object.Integer).Value
	rightValue := right.(*object.Integer).Value

	var result int64

	switch op {
	case code.OpAdd:
		result = leftValue + rightValue

	case code.OpSub:
		result = leftValue - rightValue

	case code.OpMul:
		result = leftValue * rightValue

	case code.OpDiv:
		result = leftValue / rightValue

	default:
		return fmt.Errorf("unknow integer operator: %d", op)
	}

	return vm.push(&object.Integer{Value: result})
}
```

Here is where I finally unwrap the integers contained in the left and right operands and produce a result according to the **op**. There shouldn't be any surprises here, because this method has a really similar counterpart in the **evaluator** package I built in the first book.

And now, the test pass.

Addition, subtraction, multiplication, division - they all work. As a single operations, combined, grouped by parentheses; all I do is pop operands off the stack and push the result back. Stack arithmetic. 

### Booleans
Of course, Monkey has more than four operators I added. There are also the comparison operators `==`, `!=`, `>`, `<` and the two prefix operators `!` and `-`. And without booleans I couldn't represent the results of these operators (well, except for the **-** prefix), but booleans also exist as literal expressions in Monkey: `true;`, `false;`.
I'll start by adding support for these two literals. That way I already have boolean data type in place when I add the operators.

In my **evaluator**, a boolean literal evaluates to the boolean value it designates: true or false. Now, I'm working with a compiler and a VM, so I have to adjust my expectations a little bit. Instead of boolean literals *evaluating* to boolean value, I now want them to cause the VM to load the boolean values on to the stack.
That's pretty close to what integer literals do ant those are compiled to **OpConstant** instructions. I ***could*** treat `true` and `false` as constants too, but that would be a waste, not only of bytecode but also of compiler and VM resources. Instead, I'll not define two new opcodes that directly tell the VM to push an `*oject.Boolean` on to the stack:

```go
// code/code.go

const (
	// [...]
	OpTrue
	OpFalse
)

var definitions = map[Opcode]*Definition{
	// [...]
	OpTrue:     {"OpTrue", []int{}},
	OpFalse:    {"OpFalse", []int{}},
}
```

Both opcodes have no operands, and simply tell the VM "push true or false on to the stack". With this, I can now use that to create a compiler test in which I make sure that the boolean literals **true** and **false** are translated to **OpTrue** and **OpFalse** instructions:

```go
// compiler/compiler_test.go

func TestBooleanExpressions(t *testing.T) {
	tests := []compilerTestCase{
		{
			input:             "true",
			expectedConstants: []interface{}{},
			expectedInstructions: []code.Instructions{
				code.Make(code.OpTrue),
				code.Make(code.OpPop),
			},
		},
		{
			input:             "false",
			expectedConstants: []interface{}{},
			expectedInstructions: []code.Instructions{
				code.Make(code.OpFalse),
				code.Make(code.OpPop),
			},
		},
	}

	runCompilerTests(t, tests)
}
```

```shell
go test ./compiler 
--- FAIL: TestBooleanExpressions (0.00s)
    compiler_test.go:165: testInstructions failed: _wrong instructions length.
        want="0000 OpTrue\n0001 OpPop\n"
        got="0000 OpPop\n"
FAIL
FAIL    github.com/vit0rr/mumu/compiler 0.536s
FAIL
```

It fails because the compiler only knows that it should emit an **OpPop** after expression statements. So, in order to emit **OpTrue** or **OpFalse** instructions I need to add a new **case** branch for `*ast.Boolean` to the compiler's **Compile** method:

```go
// compiler/compiler.go

func (c *Compiler) Compile(node ast.Node) error {
	switch node := node.(type) {
	// [...]

	case *ast.Boolean:
		if node.Value {
			c.emit(code.OpTrue)
		} else {
			c.emit(code.OpFalse)
		}

	// [...]
}
```

And now, test pass:
```shell
➜ go test ./compiler
ok      github.com/vit0rr/mumu/compiler 0.480s
```

Now, I need to tell the VM about **true** and **false**. And just like in the **compiler** package I now create a second test function:

```go
// vm/vm_test.go

func TestBooleanExpressions(t *testing.T) {
	tests := []vmTestCase{
		{"true", true},
		{"false", false},
	}

	runVmTests(t, tests)
}
```

Pretty similar to the `TestIntegerArithmetic`. But for booleans, I now need to create a new branch for booleans to `testExpectedObject` helper, and create a new `testBooleanObject` helper:

```go
// vm/vm_test.go

func testBooleanObject(expected bool, actual object.Object) error {
	result, ok := actual.(*object.Boolean)
	if !ok {
		return fmt.Errorf("object is not Boolean. got=%T (%+v)", actual, actual)
	}

	if result.Value != expected {
		return fmt.Errorf("object has wrong value. got=%t, want=%t",
			result.Value, expected)
	}

	return nil
}

func testExpectedObject(
	t *testing.T,
	expected interface{},
	actual object.Object,
) {
	t.Helper()

	switch expected := expected.(type) {
	case int:
		err := testIntegerObject(int64(expected), actual)
		if err != nil {
			t.Errorf("testIntegerObject failed: %s", err)
		}

	case bool:
		err := testBooleanObject(bool(expected), actual)
		if err != nil {
			t.Errorf("testBooleanObject failed: %s", err)
		}
	}
}
```

And... No, the tests not pass with this:
```shell
go test ./vm      
--- FAIL: TestBooleanExpressions (0.00s)
panic: runtime error: index out of range [-1] [recovered]
        panic: runtime error: index out of range [-1]

goroutine 22 [running]:
testing.tRunner.func1.2({0x100ea8b40, 0x140000b2018})
        /opt/homebrew/Cellar/go/1.23.2/libexec/src/testing/testing.go:1632 +0x1bc
testing.tRunner.func1()
        /opt/homebrew/Cellar/go/1.23.2/libexec/src/testing/testing.go:1635 +0x334
panic({0x100ea8b40?, 0x140000b2018?})
        /opt/homebrew/Cellar/go/1.23.2/libexec/src/runtime/panic.go:785 +0x124
github.com/vit0rr/mumu/vm.(*VM).pop(...)
        /Users/vitorsouza/Desktop/dev/mumu/vm/vm.go:122
github.com/vit0rr/mumu/vm.(*VM).Run(0x140000e5e28)
        /Users/vitorsouza/Desktop/dev/mumu/vm/vm.go:60 +0x1c0
github.com/vit0rr/mumu/vm.runVmTests(0x140000f11e0, {0x140000edf18, 0x2, 0x2?})
        /Users/vitorsouza/Desktop/dev/mumu/vm/vm_test.go:66 +0x29c
github.com/vit0rr/mumu/vm.TestBooleanExpressions(0x140000f11e0?)
        /Users/vitorsouza/Desktop/dev/mumu/vm/vm_test.go:124 +0x88
testing.tRunner(0x140000f11e0, 0x100eb6740)
        /opt/homebrew/Cellar/go/1.23.2/libexec/src/testing/testing.go:1690 +0xe4
created by testing.(*T).Run in goroutine 1
        /opt/homebrew/Cellar/go/1.23.2/libexec/src/testing/testing.go:1743 +0x314
FAIL    github.com/vit0rr/mumu/vm       0.512s
FAIL
```

it blows because I'm issue an **OpPop** after every expression statement to keep the stack clean. And when I try to pop something off the stack without first putting something on it, I get an index out of range panic.
The first step towards fixing this is to tell the VM about **true** and **false** and defining global **True** and **False** instances of them:

```go
// vm/vm.go

var True = &object.Boolean{Value: true}
var False = &object.Boolean{Value: false}
```

The reason for reusing global instances of the object.Boolean are the same as in my evaluator. That's just no-brainer in terms of perrmance because true will always be true. And it's easier makes comparisons, like `true == true`, I can just compare two pointers without having to unwrap the value they're pointing at.

Now, I need to push them on to the stack when instructed to do so:



```go
// vm/vm.go

func (vm *VM) Run() error {
	for ip := 0; ip < len(vm.instructions); ip++ {
		op := code.Opcode(vm.instructions[ip])

		// [...]

		case code.OpTrue:
			err := vm.push(True)
			if err != nil {
				return err
			}

		case code.OpFalse:
			err := vm.push(False)
			if err != nil {
				return err
			}

		// [...]
	}

	return nil
}
```

Nothing fancy here. Push the globals `True` and `False` on to the stack. That means that I'm actually pushing something on to the stack before trying to clean it up again, which means my tests don't blow up anymore:


```shell
➜ go test ./vm
ok      github.com/vit0rr/mumu/vm       0.504s
```

### Comparison operators
The four comparison operators in Monkey are: `==`, `!=`, `>` and `<`. I will now add support for all four of them by adding three new opcode definitions and supporting them in the compiler and the VM. Here they are:

```go
// code/code.go

const (
	OpConstant Opcode = iota
	OpPop
	OpAdd
	OpSub
	OpMul
	OpDiv
	OpTrue
	OpFalse
	OpEqual
	OpNotEqual
	OpGreaterThan
)

var definitions = map[Opcode]*Definition{
	OpConstant:    {"OpConstant", []int{2}},
	OpPop:         {"OpPop", []int{}},
	OpAdd:         {"OpAdd", []int{}},
	OpSub:         {"OpSub", []int{}},
	OpMul:         {"OpMul", []int{}},
	OpDiv:         {"OpDiv", []int{}},
	OpTrue:        {"OpTrue", []int{}},
	OpFalse:       {"OpFalse", []int{}},
	OpEqual:       {"OpEqual", []int{}},
	OpNotEqual:    {"OpNotEqual", []int{}},
	OpGreaterThan: {"OpGreaterThan", []int{}},
}
```

No operands because they do the work by comparing the two topmost elements on the stack. They tell the VM to pop them off and push the result back on. Just like opcodes for arithmetic operations.

And yes, theres is no `OpLessThan`. 
That's because with compilation it's possible to do a thing that are not possible with interpretation: reordering of code.

The expression `3 < 5` can be reordered to `5 > 3` without changing its result. And because it can be reordered, that's what my compiler is going to do. It will take every less-than expression, and reorder it to emit the greater-than version instead. Instructions set small, loop of my VM tighter and ofc learn the things I can do with compilation.

Let's write some tests:
```go
// compiler/compiler_test.go

func TestBooleanExpressions(t *testing.T) {
	tests := []compilerTestCase{
		{
			input:             "true",
			expectedConstants: []interface{}{},
			expectedInstructions: []code.Instructions{
				code.Make(code.OpTrue),
				code.Make(code.OpPop),
			},
		},
		{
			input:             "false",
			expectedConstants: []interface{}{},
			expectedInstructions: []code.Instructions{
				code.Make(code.OpFalse),
				code.Make(code.OpPop),
			},
		},
		{
			input:             "1 > 2",
			expectedConstants: []interface{}{1, 2},
			expectedInstructions: []code.Instructions{
				code.Make(code.OpConstant, 0),
				code.Make(code.OpConstant, 1),
				code.Make(code.OpGreaterThan),
				code.Make(code.OpPop),
			},
		},
		{
			input:             "1 < 2",
			expectedConstants: []interface{}{2, 1},
			expectedInstructions: []code.Instructions{
				code.Make(code.OpConstant, 0),
				code.Make(code.OpConstant, 1),
				code.Make(code.OpGreaterThan),
				code.Make(code.OpPop),
			},
		},
		{
			input:             "1 == 2",
			expectedConstants: []interface{}{1, 2},
			expectedInstructions: []code.Instructions{
				code.Make(code.OpConstant, 0),
				code.Make(code.OpConstant, 1),
				code.Make(code.OpEqual),
				code.Make(code.OpPop),
			},
		},
		{
			input:             "1 != 2",
			expectedConstants: []interface{}{1, 2},
			expectedInstructions: []code.Instructions{
				code.Make(code.OpConstant, 0),
				code.Make(code.OpConstant, 1),
				code.Make(code.OpNotEqual),
				code.Make(code.OpPop),
			},
		},
		{
			input:             "true == false",
			expectedConstants: []interface{}{},
			expectedInstructions: []code.Instructions{
				code.Make(code.OpTrue),
				code.Make(code.OpFalse),
				code.Make(code.OpEqual),
				code.Make(code.OpPop),
			},
		},
		{
			input:             "true != false",
			expectedConstants: []interface{}{},
			expectedInstructions: []code.Instructions{
				code.Make(code.OpTrue),
				code.Make(code.OpFalse),
				code.Make(code.OpNotEqual),
				code.Make(code.OpPop),
			},
		},
	}

	runCompilerTests(t, tests)
}
```

Of course it fails.
What I want from my compiler is to emit two instructions to get the operands of the infix operators on to the stack and then one instruction with the correct comparison opcode. 

I need to extend the `*ast.InfixExpression` branch in my **Compile** method, where I already emit the other infix operator opcodes:

```go
// compiler/compiler.go

func (c *Compiler) Compile(node ast.Node) error {
	switch node := node.(type) {
	case *ast.Program:
		for _, s := range node.Statements {
			err := c.Compile(s)
			if err != nil {
				return err
			}
		}

	case *ast.Boolean:
		if node.Value {
			c.emit(code.OpTrue)
		} else {
			c.emit(code.OpFalse)
		}

	case *ast.ExpressionStatement:
		err := c.Compile(node.Expression)
		if err != nil {
			return err
		}
		c.emit(code.OpPop)

	case *ast.InfixExpression:
		err := c.Compile(node.Left)
		if err != nil {
			return err
		}

		err = c.Compile(node.Right)
		if err != nil {
			return err
		}

		switch node.Operator {
		case "+":
			c.emit(code.OpAdd)
		case "-":
			c.emit(code.OpSub)
		case "*":
			c.emit(code.OpMul)
		case "/":
			c.emit(code.OpDiv)
		case ">":
			c.emit(code.OpGreaterThan)
		case "==":
			c.emit(code.OpEqual)
		case "!=":
			c.emit(code.OpNotEqual)
		default:
			return fmt.Errorf("unknow operador %s", node.Operator)
		}

	case *ast.IntegerLiteral:
		integer := &object.Integer{Value: node.Value}
		c.emit(code.OpConstant, c.addConstant(integer))
	}

	return nil
}
```

Support for `<` operators is still missing.

Since this the operator for which I want to reorder the operands, its implementation is an addition right at the beginning of the **case** branch for `*ast.InfixExpression`:
```go
// compiler/compiler.go

func (c *Compiler) Compile(node ast.Node) error {
	switch node := node.(type) {
	case *ast.Program:
	// [...]

	case *ast.InfixExpression:
		if node.Operator == "<" {
			err := c.Compile(node.Right)
			if err != nil {
				return err
			}

			err = c.Compile(node.Left)
			if err != nil {
				return err
			}

			c.emit(code.OpGreaterThan)
			return nil
		}

		err := c.Compile(node.Left)
		if err != nil {
			return err
		}

		err = c.Compile(node.Right)
		if err != nil {
			return err
		}

		// [...]
}
```

What I did here is to turn `<` into a special case.
I turn the order around and first compile `node.Right` and then `node.Left` in case the operator is `<`. After that I emit **OpGraterThan** opcode. I changed a less-than comparison into a grater-than comparison - while compiling. And it is working:
```shell
go test ./compiler                              
ok      github.com/vit0rr/mumu/compiler 0.488s
```

But I'm not done yet.
The goal is that it looks to VM as if there is not such thing as a `<` operator. All the VM should worry about are **OpGreaterThan** instructions. And now that we are sure my compiler only emits those, I can turn to my VM tests:
```go
// vm/vm_test.go

func TestBooleanExpressions(t *testing.T) {
	tests := []vmTestCase{
		{"true", true},
		{"false", false},
		{"1 < 2", true},
		{"1 > 2", false},
		{"1 < 1", false},
		{"1 > 1", false},
		{"1 == 1", true},
		{"1 != 1", false},
		{"1 == 2", false},
		{"1 != 2", true},
		{"true == true", true},
		{"false == false", true},
		{"true == false", false},
		{"true != false", true},
		{"false != true", true},
		{"(1 < 2) == true", true},
		{"(1 < 2) == false", false},
		{"(1 > 2) == true", false},
		{"(1 > 2) == false", true},
	}

	runVmTests(t, tests)
}
```

And it does not pass.
```shell
go test ./vm      
--- FAIL: TestBooleanExpressions (0.00s)
    vm_test.go:141: testBooleanObject failed: object is not Boolean. got=*object.Integer (&{Value:1})
    vm_test.go:141: testBooleanObject failed: object is not Boolean. got=*object.Integer (&{Value:2})
    vm_test.go:141: testBooleanObject failed: object is not Boolean. got=*object.Integer (&{Value:1})
    vm_test.go:141: testBooleanObject failed: object is not Boolean. got=*object.Integer (&{Value:1})
    vm_test.go:141: testBooleanObject failed: object is not Boolean. got=*object.Integer (&{Value:1})
    vm_test.go:141: testBooleanObject failed: object is not Boolean. got=*object.Integer (&{Value:1})
    vm_test.go:141: testBooleanObject failed: object is not Boolean. got=*object.Integer (&{Value:2})
    vm_test.go:141: testBooleanObject failed: object is not Boolean. got=*object.Integer (&{Value:2})
    vm_test.go:141: testBooleanObject failed: object has wrong value. got=false, want=true
    vm_test.go:141: testBooleanObject failed: object has wrong value. got=false, want=true
    vm_test.go:141: testBooleanObject failed: object has wrong value. got=true, want=false
    vm_test.go:141: testBooleanObject failed: object has wrong value. got=false, want=true
FAIL
FAIL    github.com/vit0rr/mumu/vm       0.526s
FAIL
```

Not that hard.
First, I add a new case branch to my VM's Run method, so it handles the new comparison opcodes:
```go
// vm/vm.go

func (vm *VM) Run() error {
	for ip := 0; ip < len(vm.instructions); ip++ {
		op := code.Opcode(vm.instructions[ip])

		switch op {
		// [...]

		case code.OpEqual, code.OpNotEqual, code.OpGreaterThan:
			err := vm.executeComparison(op)
			if err != nil {
				return err
			}

		// [...]
	}

	return nil
}
```
The `executeComparison` method looks pretty similar to the previusly added `executeBinaryOperation`:
```go
// vm/vm.go

func (vm *VM) executeComparison(op code.Opcode) error {
	right := vm.pop()
	left := vm.pop()

	if left.Type() == object.INTEGER_OBJ && right.Type() == object.INTEGER_OBJ {
		return vm.executeIntegerComparison(op, left, right)
	}

	switch op {
	case code.OpEqual:
		return vm.push(nativeBoolToBooleanObject(left == right))
	case code.OpNotEqual:
		return vm.push(nativeBoolToBooleanObject(left != right))
	default:
		return fmt.Errorf("unknow operator: %d (%s %s)", op, left.Type(), right.Type())
	}
}
```

First I pop the two operands off the stack and check their types. IF they're both integers, I'll defer to `executeIntegerComparison`. If not, I use `nativeBoolToBooleanObject` to turn the Go bools into Monkey `*object.Booleans` and push the result back on to the stack.

In other words, pop the operands off the stack, compare them, push the result ack on to the stack.

Second half of that again in `executeIntegerComparison`:
```go
// vm/vm.go

func (vm *VM) executeIntegerComparison(op code.Opcode, left, right object.Object) error {
	leftValue := left.(*object.Integer).Value
	rightValue := right.(*object.Integer).Value

	switch op {
	case code.OpEqual:
		return vm.push(nativeBoolToBooleanObject(leftValue == rightValue))
	case code.OpNotEqual:
		return vm.push(nativeBoolToBooleanObject(leftValue != rightValue))
	case code.OpGreaterThan:
		return vm.push(nativeBoolToBooleanObject(leftValue > rightValue))
	default:
		return fmt.Errorf("unknow operator: %d", op)
	}
}
```

I do not need to pop off anything anymore, but can go straight to unwrapping the integer values contained in left and right. And then, again, I compare the operands and turn the resulting bool into True or False. Here is the `nativeBoolToBooleanObject` helper:
```go
// vm/vm.go

func nativeBoolToBooleanObject(input bool) *object.Boolean {
	if input {
		return True
	}
	return False
}
```

And... Yes, that's three new methods: `executeComparison`, `executeIntegerComparison` and `nativeBoolToBooleanObject` make the tests pass:
```shell
go test ./vm
ok      github.com/vit0rr/mumu/vm       0.495s
```
