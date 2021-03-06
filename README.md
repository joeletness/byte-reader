# Python ETL sample exercise

## Summery
Read and deserialize transaction log data from a fictional proprietary binary format and marshal it into a data structure that can be used for further processing. Specific to this exercise is the ability to answer questions regarding total amounts of debit/credits, counting autopay events, and to display the balance for a specific user:

This is intended to demonstrate the following:
- Test-first TDD
- Refactoring for Readability
- Modular architecture
- Encapsulation/Separation Of Concerns
- Clean Code/Clean Architecture
- Documentation


### How it works
Clone this repo. It has no functional dependencies beyond the standard python 2.7 library. It can be run in two modes:
1. Display the full list of records and summary.
    ```bash
    $ python mps7_reader.py data.dat
    
    ```
2. Display the balance for a specific user by passing a user id.
    ```bash
    $ python mps7_reader.py data.dat 2456938384156277127
    
    ```
- __PyTest__ and __Mock__ are required if you wish to run the unit tests:
    ```bash
    $ pip install -r requirements.txt
    $ pytest
    ```
  - NOTE: Since this is an just an exercise I'm using the sample binary data as if it were a static fixure for the unit tests.

## Architecture

### Objects
- MPS7 _(controller)_
  - Responsible for collecting and storing LogEntry and User data along with accumulating aggregation info
- LogEntry _(data-model)_
  - Responsible for transforming chunks of bytes into data that can represent a log entry
- User _(data-model)_
  - Responsible for aggregating and reporting any specific user’s transaction amounts

### ETL Behavior
According to my interpretation of the given log specification, I would expect the first nine bytes of data to represent the header while the rest of the data contained all of the records in lengths of either 13 or 21 bytes depending on if the log entry was for starting/stopping autopay or debit/credit, respectively.

### Deserialization Algorithm
- Starting from the first byte of log content, consume each log until EOF:
  - IF debit/credit:
    - Read timestamp (4 bytes at start + 1)
    - Read userId (8 bytes at start + 5)
    - Read amount (8 bytes at start + 13)
    - Determine the next log entry’s start position (start + 21)
    - Create and store Record and User
  - IF auto-pay:
    - Read timestamp (4 bytes at start + 1)
    - Read userId (8 bytes at start + 5)
    - Determine the next log entry’s start position (start + 13)
    - Create and store Record and User

### Notes and Assumptions
 - According the the header, the expected count of records should be 71. However, there are actually 72. Since I can’t find any indication that my extraction algorithm is incorrect and after extensive manual inspection of the data and I haven't been able to find any indication that any particular record is in error (e.g. duplicate), ~~I’m going going to worry about it just pretend that it’s an “off by one” error systemic to the legacy system.~~ After additional clarification from the author of the exercise, it is expected that the legacy system may produce overruns and that it's acceptable to ignore them while indicating the error. This behavior has been implemented.
 - Some of the data is not what I would expect. For instance there are eight users who have simultaneous log entries with both a credit and debit in the same amount (net $0.00). This seems odd but since I don’t know anything about the origin data I’m going to assume that it’s just fine.
 - According to the specification the data is a log. I would have expected the log to be in chronological order but it is not. Again, since I don’t know anything about how this data was serialized, I’m going to assume that that’s perfectly normal.

### Questions/Answers
1. _What is the total amount in dollars of debits?_ __$18203.69__
2. _What is the total amount in dollars of credits?_ __$9366.00__
3. _How many autopays were started?_ __Ten__
4. _How many autopays were ended?_ __Eight__
5. _What is balance of user ID 2456938384156277127?_ __$0.00__

## Original exercise spec
```
Parse a custom protocol format
==============================

Your application must interface with an old-school mainframe MPS7 for payment
processing. This means consuming a proprietary binary protocol format.

Task
----

You must read in a file, `data.dat`, and parse it according to the
specification in Notes below.

You must answer the following questions:

* What is the total amount in dollars of debits? 
* What is the total amount in dollars of credits? 
* How many autopays were started?
* How many autopays were ended?
* What is balance of user ID 2456938384156277127?

You must supply your source code as part of your answer. Write your code in your
best programming language.

Notes
-----

MPS7 transaction log specification:

Header:

| 4 byte magic string "MPS7" | 1 byte version | 4 byte (uint32) # of records |

Record:

| 1 byte record type enum | 4 byte (uint32) Unix timestamp | 8 byte (uint64) user ID |

Record type enum:

* 0x00: Debit
* 0x01: Credit
* 0x02: StartAutopay
* 0x03: EndAutopay

For Debit and Credit record types, there is an additional field, an 8 byte
(float64) amount in dollars, at the end of the record.

All multi-byte fields are encoded in network byte order.
```