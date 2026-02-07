# Using sqltype(clob) with pointers in RPGLE

## Problem

In RPGLE, `varchar(1048576:4)` cannot be used as an SQL host variable in embedded
SQL. The SQL precompiler only supports `varchar` with a 2-byte length prefix
(max 65535) or `sqltype()` variables. However, `sqltype()` cannot be used on
external (RPGLE) procedure parameters.

This means you need a local `sqltype(clob)` variable to bridge between an RPG
parameter and embedded SQL. Copying 1MB of data into a local variable is
wasteful, so using a `based()` pointer to overlay the CLOB on the parameter
memory is the ideal approach.

## What doesn't work: `based()` on a `dcl-s` with `sqltype()`

```rpgle
dcl-s jsonUtf8 sqltype(clob:1048576) ccsid(*utf8) based(pJson);
dcl-s pJson pointer;
```

The SQL precompiler expands `sqltype(clob)` on a `dcl-s` into a `dcl-ds`,
and the `based()` keyword is **silently dropped**:

```rpgle
// Compiler-generated expansion (based keyword lost!)
DCL-DS JSONUTF8;
  JSONUTF8_LEN  UNS(10);
  JSONUTF8_DATA CHAR(1048576) CCSID(1208);
END-DS JSONUTF8;
dcl-s pJson pointer;
```

Setting `pJson = %addr(bb_jsoninput)` has no effect because the DS is not
based on `pJson`. The pointer variable exists but is disconnected from the
CLOB data structure.

## What works: `sqltype()` inside an explicit `based()` data structure

```rpgle
dcl-ds json_clob based(json_clob_p);
  json_data sqltype(clob:1048576) ccsid(*utf8);
end-ds;
dcl-s json_clob_p pointer;
```

The `based()` keyword is on the `dcl-ds` itself, so the precompiler preserves
it. The compiler expands the CLOB subfield using `OVERLAY`:

```rpgle
dcl-ds json_clob based(json_clob_p);
  // MYCLOB SQLTYPE(CLOB:1048576) CCSID(*UTF8);
  MYCLOB      CHAR(1048580) CCSID(*HEX);
  MYCLOB_LEN  UNS(10)       OVERLAY(MYCLOB);
  MYCLOB_DATA CHAR(1048576)  OVERLAY(MYCLOB:5) CCSID(1208);
end-ds;
dcl-s json_clob_p pointer;
```

The memory layout (4-byte length prefix + data) matches `varchar(n:4)`,
so pointing it at a varchar parameter works correctly:

```rpgle
json_clob_p = %addr(bb_jsoninput);
exec sql SET BBWEBOBJ.JSONINPUT = :json_data;
```

## Full example

### RMAPIGENR.sqlrpgle

This is main program that shows how to handle the SQL CLOB parameters using pointers.

```rpgle
**free

//==============================================================================
// RMAPIGENR  - Generic API Wrapper
//
// Called via external procedure RMAPIGEN.
//==============================================================================

ctl-opt dftactgrp(*no);
ctl-opt option(*srcstmt:*nodebugio);
ctl-opt main(RMAPIGENR);
ctl-opt text('Generic API Wrapper');

// Template for a 1M UTF-8 CLOB
dcl-s JSON_CLOB_1M_t varchar(1048576:4) ccsid(*utf8) template;

dcl-proc RMAPIGENR;
  dcl-pi *n;
    in_api_program char(10) const;
    in_jsoninput like(JSON_CLOB_1M_t);
    out_jsonoutput like(JSON_CLOB_1M_t);
  end-pi;

  dcl-s APIPGM char(10);
  dcl-pr API extpgm(APIPGM) end-pr;

  // Pointer-based CLOB overlay - avoids copying 1MB of data
  dcl-ds json_clob based(json_clob_p);
    json_data sqltype(clob:1048576) ccsid(*utf8);
  end-ds;
  dcl-s json_clob_p pointer;

  exec sql set option commit = *none, closqlcsr= *endmod, naming = *sys;

  exec sql CREATE OR REPLACE VARIABLE JSONINPUT CLOB(1M) CCSID 1208;
  exec sql CREATE OR REPLACE VARIABLE JSONOUTPUT CLOB(1M) CCSID 1208;

  // Point CLOB at input parameter and set the SQL global variable
  json_clob_p = %addr(in_jsoninput);
  exec sql SET JSONINPUT = :json_data;

  // Call the API program
  // - The API program takes its input from the SQL global variable JSONINPUT
  //   and writes its output to another SQL global variable JSONOUTPUT.
  APIPGM = in_api_program;
  API();

  // Point CLOB at output parameter and read the SQL global variable
  json_clob_p = %addr(out_jsonoutput);
  exec sql SET :json_data = JSONOUTPUT;

  return;
end-proc;
```

### RMAPIGEN.sql

This is a simple SQL external procedure over `RMAPIGENR`.

```sql
--------------------------------------------------------------------------------
-- RMAPIGEN   - Generic API Wrapper
--
-- External procedure pointing to RMAPIGENR.
--------------------------------------------------------------------------------

-- 1. API_PROGRAM     CHAR(10)     INPUT   - API program to call
-- 2. JSONINPUT       CLOB(1M)     INPUT   - JSON request data
-- 3. JSONOUTPUT      CLOB(1M)     OUTPUT  - JSON response data

CREATE OR REPLACE PROCEDURE RMAPIGEN (
    IN     API_PROGRAM CHAR(10) ,
    IN     JSONINPUT   CLOB(1M) CCSID 1208 ,
    OUT    JSONOUTPUT  CLOB(1M) CCSID 1208
)

LANGUAGE RPGLE
PARAMETER STYLE SQL
DETERMINISTIC
MODIFIES SQL DATA
CALLED ON NULL INPUT
PROGRAM TYPE MAIN
EXTERNAL NAME RMAPIGENR
SPECIFIC RMAPIGEN;

LABEL ON PROCEDURE RMAPIGEN IS 'Generic API Wrapper' ;
```

### RMHELLO.sqlrpgle

This is a sample program that provides a simple example of how this all fits together.

```rpgle
**free

//==============================================================================
// RMHELLO    - Sample API Program
//
// Called via external procedure RMAPIGEN.
//==============================================================================

ctl-opt dftactgrp(*no);
ctl-opt option(*srcstmt:*nodebugio);
ctl-opt main(RMHELLO);
ctl-opt text('Sample API Program');

dcl-proc RMHELLO;

  exec sql set option commit = *none, closqlcsr= *endmod, naming = *sys;

  dcl-s json_data sqltype(clob:1048576) ccsid(*utf8);
  dcl-s user_name varchar(50) ccsid(*utf8);

  // Input parameters are held in the SQL global variable JSONINPUT
  exec sql SET :json_data = JSONINPUT;

  exec sql
    SELECT USER_NAME INTO :user_name FROM
      JSON_TABLE(:json_data, '$' COLUMNS(USER_NAME VARCHAR(50) PATH '$.name'));

  exec sql
    SET :json_data = JSON_OBJECT('message' : 'Hello, ' || :user_name || '!');

  // Output parameters are returned in the SQL global variable JSONOUTPUT
  exec sql SET JSONOUTPUT = :json_data;

  return;
end-proc;
```

The sample program can then be called with the following SQL statement.

```sql
CALL RMAPIGEN (
  API_PROGRAM => 'RMHELLO',
  JSONINPUT   => '{ "name":"Richard" }',
  JSONOUTPUT   => ?
);

-- Output Parameter #3 (JSONOUTPUT) = {"message":"Hello, Richard!"}
```

## Key takeaway

When using `sqltype()` with `based()` pointers, always place the `sqltype()`
field **inside an explicit `dcl-ds`** that carries the `based()` keyword.
Never use `based()` directly on a `dcl-s` with `sqltype()` as the precompiler
will silently discard it.
