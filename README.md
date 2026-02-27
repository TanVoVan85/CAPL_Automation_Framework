# CAPL_Automation_Framework
We are building a reusable CAPL function library for ECU automation testing in Vector CANoe/CANalyzer.
In previous sessions, we created and updated these files:
1. `can_utils.cin` - CAN helper functions.
2. `uds_diag.cin` - UDS request/response handling, wrappers, target-name overload, and bounded retry handling for NRC 0x78.
3. `test_utils.cin` - Logging, verdict, and timing helpers.
4. `flash_utils.cin` - Security access and routine control helpers.
5. `ECU_config_detail.cin` - ECU IDs, addressing, TP settings, and ISO-TP network timing parameters (As, Ar, Bs, Br, Cs, Cr).
6. `test_example.can` - Example CAPL testcases intended for XML Test Module usage.

Please help me implement the next phase of the library by completing the following tasks:

### 1: Implement `.can` Testcases for XML Test Module (No MainTest)
Update or create `.can` testcase files that are compatible with a CANoe CAPL XML Test Module already created in the project.
- Do **not** use `void MainTest()`.
- Use only `testcase` functions so they can be called/imported by XML test configuration.
- Include at least:
	- one diagnostic session testcase,
	- one tester present testcase,
	- one DID read testcase.
- Use IDs/channel from `ECU_config_detail.cin` (no hard-coded IDs in testcase logic).

### 2: Improve Robustness of UDS Test Flow
Review `uds_diag.cin` and ensure loops cannot run infinitely.
- Keep bounded retry behavior for NRC `0x78` and unexpected frames.
- Return explicit status when retry limit is exceeded.

### 3: Update Prompt
As a final step, generate an updated version of this prompt for the next session.

### 4: Catching common error while using AI to generate script CAPL, fixing and improve
When the request is not explicit, these are the 4 highest-value questions:
1) Is this CAPL running in a **Test Module** (`testcase`, `TestWaitFor...`) or a **CAPL program node**?
2) Which APIs are available in your environment? (Open **Help → CAPL Browser** and confirm prototypes.)
3) What transport behavior is required?
   - Raw CAN only, or ISO-TP (SF + FF/CF + FC)?
4) Maximum payload sizes?
   - ISO-TP max is 4095 bytes, but do you want to buffer that in CAPL?

If answers are unknown, default to:
- Minimal API usage
- Globals in `variables {}`
- ISO-TP lengths bounded and checked
5) CAPL Syntax & Semantics Cheat Sheet checking:
-  avoids local-stack limitations and local-array parse issues
-  state, flags, and large buffers under variabble {}
6) Avoid non-constant initializers
 Many CAPL compilers are stricter than C.
 Example:   
    **Prefer:**
    ```capl
    byte x;
    x = someArray[i];
    ```
    **Over:**
    ```capl
    byte x = someArray[i];
    ```
7) Avoid local array declarations
8) Pointers and dynamic allocation:
- Treat pointers as **not allowed** unless you can prove they compile in your context.
- Avoid dynamic allocation completely.
9) Avoid reserved identifiers
- Some words are reserved by CAPL (keywords/types). Using them as variable or parameter names can trigger confusing parse errors.
  
  **Example (bad):**
  ```capl
  void test_logError(char message[])
  {
    write("ERROR: %s", message);
  }
  ```
  
  **Fix:** rename the identifier to something safe.
  ```capl
  void test_logError(char msgText[])
  {
    write("ERROR: %s", msgText);
  }
  ```
  
  ---  


### Requirements:
- Maintain existing coding style with detailed comments (Purpose, Inputs, Returns).
- Use the defined error codes (`UDS_OK`, `UDS_ERR_TIMEOUT`, `UDS_ERR_NRC`, `UDS_ERR_PARAM`).
- Provide CAPL code in fenced ```capl code blocks.
- Briefly explain design choices before code.
- Checking and verify to avoid common error in CAPL syntax and rules 
