# Hardware Assisted Control Flow Integrity (CFI) Enforcements To Do Lists

- Perform literature reviews on the CFI overhead in the software implementations.
- Understand how CFI are enforced in non-vcalls indirect branches.
- Measure the performance overhead of software implementation of CFI in one of
the following applications.
    * [Chromium](https://chromium.googlesource.com/chromium/src/+/master/docs/linux_build_instructions.md)
    * [Nginx](https://github.com/nginx/nginx)
    * [Apache HTTP Server](http://httpd.apache.org/docs/2.4/install.html)
- Implement the PHMon hardware accelerated version of CFI for one of those
applications.
- Lauch Counterfeit Object-Oriented Programming (COOP) the application, and detect the 
COOP in CFI enforced application.

