# Using sqltype(clob) with pointers in RPGLE

## Problem

In RPGLE, `varchar(1048576:4)` cannot be used as an SQL host variable in embedded
SQL. The SQL precompiler only supports `varchar` with a 2-byte length prefix
(max 65535) or `sqltype()` variables. However, `sqltype()` cannot be used on
procedure parameters.

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

```rpgle
**free

// Template for 1M UTF-8 varchar parameters
dcl-s JSON_CLOB_1M_t varchar(1048576) ccsid(*utf8) template;

dcl-proc RMAPIGENW;
  dcl-pi *n;
    bb_program    char(10) const;
    bb_jsoninput  like(JSON_CLOB_1M_t);
    bb_jsonoutput like(JSON_CLOB_1M_t);
  end-pi;

  dcl-s BBAPIPGM char(10);
  dcl-pr BBAPI extpgm(BBAPIPGM) end-pr;

  // Pointer-based CLOB overlay - avoids copying 1MB of data
  dcl-ds json_clob based(json_clob_p);
    json_data sqltype(clob:1048576) ccsid(*utf8);
  end-ds;
  dcl-s json_clob_p pointer;

  exec sql set option commit = *none, closqlcsr= *endmod, naming = *sys;

  // Point CLOB at input parameter and set the SQL global variable
  json_clob_p = %addr(bb_jsoninput);
  exec sql SET BBWEBOBJ.JSONINPUT = :json_data;

  // Call the program
  BBAPIPGM = bb_program;
  BBAPI();

  // Point CLOB at output parameter and read the SQL global variable
  json_clob_p = %addr(bb_jsonoutput);
  exec sql SET :json_data = BBWEBOBJ.JSONOUTPUT;

  *inlr = *on;
  return;
end-proc;
```

## Key takeaway

When using `sqltype()` with `based()` pointers, always place the `sqltype()`
field **inside an explicit `dcl-ds`** that carries the `based()` keyword.
Never use `based()` directly on a `dcl-s` with `sqltype()` as the precompiler
will silently discard it.
