# Qemu Insides
Insides of the Qemu software, version 4.1 (stable), at [https://github.com/qemu/qemu/tree/stable-4.1](https://github.com/qemu/qemu/tree/stable-4.1). This repo is inspired by [0xAX/linux-insides](https://github.com/0xAX/linux-insides) which gave me a really good introduction to the Linux Kernel. I'm also intersted in other virtualization techniques (that are related to Qemu) like Xen and KVM. Consider making me a donation and starring this repo so I have the strength to make new repos explaining their insides too :).

# Donations

Bitcoin: 182ZMwHHYg8X6f345675pCroVod2tMaGmH
Ethereum: 0x94503E5a4de04575732606eeE2Dd6D404FF1c9e9
Paypal: eu@lucaszanella.com

# How to read

You should begin at `Qemu/1-qemu-init.md`. While reading, you can click on links that send you rigth on the function being talked about, in the exact tag (stable-4.1) on github. However, for better reading the code I suggest using VSCode and installing the Microsoft C++ Intellisense extension `ms-vscode.cpptools`. It enables you to rigth click on functions and variables and peek or go directly to its position on its file. Don't forget to change the branch to `stable-4.1` before you start reading, otherwise you're gonna find some different things (in the master branch).

VSCode tips:

* Rigth click on function name and peek/go to definition
* Control F for searching in the same page
* Control shift F for searching throughout all files (VERY useful)
* Using Control Shift F with regex and/or enabling Match Case ir order to find specific or generic names
* Finding random functions that are interesting and them backtrace their calls by navigating with "rigth click + go to definition" technique

You migth also use VSCode to debug qemu by adding breakpoints and analyzing backtrace to see where a function is called (TODO: explain how to do this).

# How to contribute?

I'm very open to contributions. I don't understand all the code, and I'm doing this mainly to learn in the process. If you find ANY mistakes, please contact me or open a pull request. If you want to contribute with a new chapter, I'm also thankful. Since I'm not going to cover the code for every emulated device (because there are MANY), it should be nice if different people covered different devices. We migth even create a separate folder for explaining different device emulations.

Please also consider that english is not my main language, I'm from Brazil.