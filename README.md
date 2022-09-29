# coprocessorjslib
coprocessorjslib is a C library for Classic Macintosh systems for sending programs and commands over their serial ports with https://github.com/CamHenlin/coprocessor.js.

coprocessorjslib is intended to be included in other C/C++ programs targeted at Classic Macintosh systems, allowing them to easily hand off JavaScript workloads over the serial port.

## Terminology
- `Classic Macintosh` - Pre-OS X Macintosh OS. Currently testing with System 6.0.8 but older versions probably work fine too
- `host machine` or `machine at the other end of the serial port` - a computer connected to the serial port of the Classic Macintosh, capable of running nodejs programs. Already running https://github.com/CamHenlin/coprocessor.js.

## How do I use it?

Include `coprocessorjs.h` and `coprocessorjs.c` in your Retro68, MPW, THINK C, etc program.

From there, there are 5 functions and 1 variable available to your C code:

- setupCoprocessor
- sendProgramToCoprocessor
- callFunctionOnCoprocessor
- callVoidFunctionOnCoprocessorAsync
- callEvalOnCoprocessor
- coprocessorEventLoopActions
- closeSerialPort
- MAX_RECEIVE_SIZE
- system7OrGreater
- asyncCallActive

#### setupCoprocessor
`void setupCoprocessor(char *applicationId, const char *serialDeviceName)`

Used for setting up the coprocessorjslib, setting up serial ports, etc. This must be called before other coprocessorjslib calls.

- *applicationId*: unique string for your application, could be something like "my test application v0.01"
- *serialDeviceName*: "modem" or "printer" - whichever serial port you want to use

#### sendProgramToCoprocessor
`void sendProgramToCoprocessor(char* program, char *output)`

This function is used for sending programs from the Classic Macintosh application to the host. This function expects a string containing a full nodejs app. The machine at the other end of the serial port running https://github.com/CamHenlin/coprocessor.js will set up the application its end, call npm install, etc., and return a success/failure message to the output variable. This should be the 2nd call that your program makes to coprocessorjslib, after `setupCoprocessor`.

- *program*: the full text of a nodejs application, delimited in a custom format. See here: https://github.com/CamHenlin/retro68-coprocessorjs-test/blob/main/compile_js.sh for example of how to compile a complete multi-file nodejs program to a single string in the proper format. Additionally, see here for format reference: https://github.com/CamHenlin/coprocessor.js/blob/0a8d3d16f093477068d6be23ab9adc36d845aacf/README.md#now-send-things-over-the-serial-port
- *output*: the result of the program installation on the remote device will be set here. `"SUCCESS"` is the expected result

#### callFunctionOnCoprocessor
`void callFunctionOnCoprocessor(char* functionName, char* parameters, char* output)`

Used for calling functions on the nodejs application that the Classic Macintosh application sent to the host and getting results back. This can be called any time after `sendProgramToCoprocessor` and can be called repeatedly. This function is the main purpose behind coprocessorjslib.

- *functionName*: unique string for your application, could be something like "my test application v0.01"
- *parameters*: function parameters for your nodejs function, each parameter should be delimited by the string `&&&`. See here for format reference: https://github.com/CamHenlin/coprocessor.js/blob/0a8d3d16f093477068d6be23ab9adc36d845aacf/README.md#now-send-things-over-the-serial-port if you are curious about this
- *output*: the return result of the function call from the remote device will be set here.

#### callVoidFunctionOnCoprocessorAsync
`void callVoidFunctionOnCoprocessorAsync(char* functionName, char* parameters)`

Used for asynchronously calling functions on the nodejs application that the Classic Macintosh application sent to the host. `callVoidFunctionOnCoprocessorAsync` will never return results to the program. Instead, results should be retrieved with a later `callFunctionOnCoprocessor` call. This can be called any time after `sendProgramToCoprocessor` and can be called repeatedly. This function is the main purpose behind coprocessorjslib. Requires that you insert a call to `coprocessorEventLoopActions` in your event loop to ensure that stacked `callVoidFunctionOnCoprocessorAsync` are drained.

- *functionName*: unique string for your application, could be something like "my test application v0.01"
- *parameters*: function parameters for your nodejs function, each parameter should be delimited by the string `&&&`. See here for format reference: https://github.com/CamHenlin/coprocessor.js/blob/0a8d3d16f093477068d6be23ab9adc36d845aacf/README.md#now-send-things-over-the-serial-port if you are curious about this

#### coprocessorEventLoopActions
`void coprocessorEventLoopActions()`

This function should be called in your event loop if your program intents to use `callVoidFunctionOnCoprocessorAsync`. If you do not do this, your serial port will not be able to drain repeated asynchronous function calls and will also may lose async function calls entirely if there is a synchronous function call happening when `callVoidFunctionOnCoprocessorAsync` is called. See https://github.com/CamHenlin/FocusedEdit/blob/d00bb6f9d6f9940263e3408d73c1a79c02515ec2/TESample.c#L389 for reference.

#### callEvalOnCoprocessor
`void callEvalOnCoprocessor(char* toEval, char* output)`

Evaluate JS against the loaded program on the host machine and get a result back. This can be called any time after `sendProgramToCoprocessor` and can be called repeatedly. 

- *callEvalOnCoprocessor*: raw JS code to eval
- *output*: the return result of the function call from the remote device will be set here. 

#### callEvalOnCoprocessor
`OSErr closeSerialPort()`

Used at the end of your program once you no longer expect any serial communication between your Classic Macintosh program and the machine at the other end of the serial port

#### MAX_RECEIVE_SIZE
`int MAX_RECEIVE_SIZE`

This is the maximum size that coprocessorjslib is set up to receive in a single function call. In your code, you should expect results to be up to this size and handle appropriately.

#### isSystem7OrGreater
`Boolean isSystem7OrGreater`

Check that is set during setupCoprocessor to tell us if we are running on System 7 or greater. This may be important to our program because it causes the serial port to behave differently.

#### asyncCallActive
`Boolean asyncCallActive`

Boolean set when an async call is active via `callVoidFunctionOnCoprocessorAsync`. This is available so we can prevent ourselves from making a subsequent call (if desired) while we are still running a call.

## example C code
Also available in more runnable code at https://github.com/CamHenlin/retro68-coprocessorjs-test/.

Example JS code is here: https://github.com/CamHenlin/retro68-coprocessorjs-test/tree/c7e7b4066fbf86a8f021b2be71c11c9ec64b1e87/JS and see above note on `sendProgramToCoprocessor` for packaging info. 

include `coprocessorjs.h` and `coprocessorjs.c` in your project to get started

```
#include <Resources.h>
#include <stdio.h>
#include "output_js.h" // see note under sendProgramToCoprocessor above about how the JS is packaged
#include "coprocessorjs.h" // from this repo!

int main(int argc, char** argv) {

    setupCoprocessor("my_application_id", "modem"); // 1st param can be any string we want, 2nd could also be "printer", for testing in PCE: modem is 0 in PCE settings - printer would be 1

    char programResult[MAX_RECEIVE_SIZE];

    // send the nodejs program from our C program to the coprocessorjs service
    // note that this example only has a single JS function "getBodyAtURL" which gets the visible text from a webpage at a specific URL
    sendProgramToCoprocessor(OUTPUT_JS, programResult); // we would expect programResult to be "SUCCESS" if loading our application on coprocessorjs worked

    char jsFunctionResponse[MAX_RECEIVE_SIZE];

    // call the getBodyAtURL JS function on the coprocessorjs host. have it get the main page contents for the main coprocessorjs git repo and put it in jsFunctionResponse
    callFunctionOnCoprocessor("getBodyAtURL", "https://github.com/CamHenlin/coprocessor.js", jsFunctionResponse);

    printf("CoprocessorJS Test App: getBodyAtURL function call response:\n");
    printf(jsFunctionResponse);
    printf("\n");

    printf("\ntest complete - process any key to quit\n");
    getchar();
    closeSerialPort();

    return 0;
}
```

## What's next?
This is still in somewhat early stages but is enough to start building other applications against to see where it goes. Here's a short list of things that would be nice:

- detection of available serial ports
- user-selectable speeds
- basic compression (pretty sure LZ4 is viable on a 68k)

## Questions? Comments? Using the library?
Open an issue, open a PR, shoot me a message - I'd love to hear about what you're building with coprocessorjs.

## Programs that use Coprocessor
- https://github.com/CamHenlin/FocusedEdit
- https://github.com/CamHenlin/MessagesForMacintosh 