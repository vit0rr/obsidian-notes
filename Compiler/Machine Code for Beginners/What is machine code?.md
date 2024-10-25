When you give a computer a program in some language, like Rust, all the instructions and data are at some point translated into machine code inside the computer. It's how the computer will understand the instructions that you wrote in a more human-readable way.

In machine code, each instruction and piece of information is represented by binary number - it's a number system with only 1s, and 0s. And inside the computer, these binary numbers are represented by pulses of electricity, with. a pulse for a 1, and no pulse for a 0. The pulses and no-pulses are called "bits". 
As you probably know, a byte, is a group of 8 bits. Each byte of pulses and no-pulses represents the binary number for one instruction or piece information in machine code.

Of course we do not write machine code. The translation to machine code are made by a program called **interpreter**. This program translate your commands like adding two numbers - that you can read and understand, to machine code - that the computer can read and understand.
![[4-interpreter-image.png]]

The term machine code is also used to refer to programs written in a form which is much closer to the computer's code, than JavaScript or Rust. In a machine code program you have to give the computer all the separate instructions it needs to carry out a task such adding two numbers.

## Programming in machine code
There are several different ways of writing machine code programs.
You could write all the instructions in binary numbers, you could use another number system called hex, a short for hexadecimal, or using assembly language.
In Assembly Language each instruction to the computer is represented by a mnemonic.
![[assembly-example.png]]
Left, is a part of a machine code program in hex. The hex numbers system has sixteen digits and uses the symbol 0-9 and A-F to represent the numbers 0 to 15. The hex number at the beginning of each line of the program is an instruction (3E).

On right, is the same program but in assembly language. The mnemonic LD A (load A), means the same as hex number 3E. In both these programs, each line contains an instruction which is the equivalent of a single instruction in the computer's own code.