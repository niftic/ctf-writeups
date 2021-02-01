# salt2
This challenge was the only PWN challenge of the Hackfest 2020 CTF. Our team was the only one to solve it in time.

## Recon
The binary is a costume shop. You can buy costumes, see the bill, edit your choices or give a coupon or a feedback.

The binary is 64 bits, with PIE protection but (oddly) no NX.

We notice that negatives indexes can be used while choosing a costume and editing our cart.

We tried to leak interesting values from the billing functionality by buying costumes with negatives indexes. The available costumes are stored in an array in the `.bss`.
~~~c
struct costume {
    char[28] name;
    int price;
    char[500] description;
}
~~~
We can see that every 532 bytes, we can leak a single `int` value by checking the resulting bill. We targeted PLT and GOT entries, but unfortunately, nothing interesting was at the right offset.

The cart editing was more promising. Our cart is simply an array of integers and it's passed to the functions by value. Therefore, it is stored on the stack and the negative indexes allow us to tamper with stack values as `int`. The first thing that comes to mind is to create a simple ROP chain with this primitive. However, with PIE and ASLR we don't have anywhere to return to.

Finally, we notice a format string in the feedback function. The problem (we tought at the moment) is that only 7 characters are allowed, and the first one must be a `G`. This only grants us access to 3 values on the stack with the string `G%x%x%x`. Fortunately, the first value is an address that points to the stack.

## The plan
We can use the 4 bytes write primitive to write a shellcode somewhere on the stack, then overwrite the return address and jump to it. The problem is that we can't overwrite the return address in one shot since we can only write 4 bytes at a time. After a while, we decide to use the `leave;ret` gadget at the end of the `main` function to help us. We can write near our shellcode a fake stack which will contain the address of the shellcode as a return address. We can then modify the saved rbp of the edition function to point next to this pivot. When the function returns, rbp will point to the pivot and the next `leave` instruction will allow us to control the stack and thus the return address and the execution flow.

## The workaround
We discover that the exploit works locally, but not remotely... what a surprise. We highly suspect a difference in the offsets due to different environment variables since we're playing on the stack. To find the correct offset without much hassle, we modify the script to accept it as an argument and we use a simple bash loop to iterate over nearby values.
~~~bash
for i in $(seq -10 10); do echo $i; ./exploit.py $i; done
~~~
And just like that, we magically obtain a shell with offset 4.

## The better way
Turns out I didn't do my homework! It seems C++11 introduced new operators in format strings. The `$` symbol can be used to specify directly which argument is to be used in the format. Using the format string `%11$x`, we could skip over values on the stack and leak a PIE address. The intended solution was to store the shellcode in the `.bss` using the coupon functionality and jump to it directly by overwriting the lower 4 bytes of the return address of the edition function. This is possible since the higher 4 bytes of the `.text` and `.bss` are the same.

Whatever works...

Overall this was a very fun challenge. Kudos to the author.

*- niftic*