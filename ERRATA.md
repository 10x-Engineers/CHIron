# #1 Initial state UCE of ReadOnce

## 1.1 Background

In AMBA Specification Issue E.c and eariler, **initial state UCE** (at the sending of Read request) **was allowed for ReadOnce** (also ReadOnceCleanInvalid, ReadOnceMakeInvalid).   
But UCE (or any state with Unique) obtains the **possible silent transition of Unique Clean to Unique Dirty**, which violates the initial state constraint of ReadOnce.  
This rule was "corrected" in AMBA CHI Specification Issue G, constrainting the initial state of ReadOnce to only I.

## 1.2 Erratum

In AMBA CHI Specification Issue E.c and earlier issues:

| Request | UD | UC | SD | SC | I | UDP | UCE |
|-----------|---|---|---|---|---|---|---|
| ReadOnce | - | - | - | - | Y | - | **Y** |
| ReadOnceCleanInvalid | - | - | - | - | Y | - | **Y** |
| ReadOnceMakeInvalid | - | - | - | - | Y | - | **Y** |
> Table. Permitted Requester cache state at the sending on Read request

In AMBA CHI Specification Issue G:

| Request | UD | UC | SD | SC | I | UDP | UCE |
|-----------|---|---|---|---|---|---|---|
| ReadOnce | - | - | - | - | Y | - | **-** |
| ReadOnceCleanInvalid | - | - | - | - | Y | - | **-** |
| ReadOnceMakeInvalid | - | - | - | - | Y | - | **-** |
> Table. Permitted Requester cache state at the sending on Read request

In CHIron, **we always obey the specification of Issue G** for any version of issue configuration.

This erratum was done by the following change:

```diff cpp
--- a/chi/xact/chi_xact_state/cst_xact_read.hpp
+++ b/chi/xact/chi_xact_state/cst_xact_read.hpp
@@ -39,40 +39,40 @@ namespace CHI {
                     ReadNoSnp_I
                 };
                 // ----------------------------------------------------------------------------------------------------
-                // ReadOnce             | I, UCE        | -          | I     | CompData_UC,   | RespSepData + 
+                // ReadOnce             | I             | -          | I     | CompData_UC,   | RespSepData + 
                 //                      |               |            |       | CompData_I     | DataSepResp_UC
-                constexpr CacheStateTransition ReadOnce_I_UCE_to_I = {
-                    CacheStateTransition(I | UCE, I).TypeRead()
+                constexpr CacheStateTransition ReadOnce_I_to_I = {
+                    CacheStateTransition(I, I).TypeRead()
                         .CompData(I | UC)
                         .DataSepResp(UC)
                 };
                 //
                 constexpr std::array ReadOnce = {
-                    ReadOnce_I_UCE_to_I
+                    ReadOnce_I_to_I
                 };
                 // ----------------------------------------------------------------------------------------------------
-                // ReadOnceCleanInvalid | I, UCE        | -          | I     | CompData_UC,   | RespSepData + 
+                // ReadOnceCleanInvalid | I             | -          | I     | CompData_UC,   | RespSepData + 
                 //                      |               |            |       | CompData_I     | DataSepResp_UC
-                constexpr CacheStateTransition ReadOnceCleanInvalid_I_UCE_to_I = {
-                    CacheStateTransition(I | UCE, I).TypeRead()
+                constexpr CacheStateTransition ReadOnceCleanInvalid_I_to_I = {
+                    CacheStateTransition(I, I).TypeRead()
                         .CompData(I | UC)
                         .DataSepResp(UC)
                 };
                 //
                 constexpr std::array ReadOnceCleanInvalid = {
-                    ReadOnceCleanInvalid_I_UCE_to_I
+                    ReadOnceCleanInvalid_I_to_I
                 };
                 // ----------------------------------------------------------------------------------------------------
-                // ReadOnceMakeInvalid  | I, UCE        | -          | I     | CompData_UC,   | RespSepData + 
+                // ReadOnceMakeInvalid  | I             | -          | I     | CompData_UC,   | RespSepData + 
                 //                      |               |            |       | CompData_I     | DataSepResp_UC
-                constexpr CacheStateTransition ReadOnceMakeInvalid_I_UCE_to_I = {
-                    CacheStateTransition(I | UCE, I).TypeRead()
+                constexpr CacheStateTransition ReadOnceMakeInvalid_I_to_I = {
+                    CacheStateTransition(I, I).TypeRead()
                         .CompData(I | UC)
                         .DataSepResp(UC)
                 };
                 //
                 constexpr std::array ReadOnceMakeInvalid = {
-                    ReadOnceMakeInvalid_I_UCE_to_I
+                    ReadOnceMakeInvalid_I_to_I
                 };
                 // ----------------------------------------------------------------------------------------------------
                 // ReadClean            | I             | -          | SC     | CompData_SC    | RespSepData + 
```
# #2 DoNotGoToSD of SnpQuery

## 2.1 Background

AMBA CHI Specification Issue E.b contradicts itself on the `DoNotGoToSD` field of `SnpQuery`.

*Do not transition to SD state, DoNotGoToSD* on page 13-434 lists `SnpQuery` under
**"Inapplicable, and must be set to zero in"**, while Table A-10 on page A-490 still carries `1`
("Applicable. Field value is used, must be set to one", Table A-1 on page A-482).

Appendix D's Issue E.b change list records *"Correction: Requirements for DoNotGoToSD in SnpQuery
updated"* against page 13-434, so §13.10.34 is the edited clause and Table A-10 was not
resynchronised. Table 13-30's *"If already in SD state, then except for SnpQuery must exit SD
state"* carve-out is unreachable under §13.10.34 for the same reason.

Zero is also the only value that agrees with the semantics of `SnpQuery`: §4.8.2 on page 4-228
requires the Snoopee not to change cache state, so a bit whose sole effect is to force an SD
Snoopee out of SD has nothing to act on.

## 2.2 Erratum

In Table A-10 on page A-490:

| Snoop Request message | DoNotGoToSD |
|---|---|
| SnpQuery | **1** |
> Table. Snoop Request message field mappings

Per *Do not transition to SD state, DoNotGoToSD* on page 13-434:

| Snoop Request message | DoNotGoToSD |
|---|---|
| SnpQuery | **0a** |
> Table. Snoop Request message field mappings

In CHIron, **we obey §13.10.34 over Table A-10**.

This erratum was done by the following change:

```diff cpp
--- a/chi/xact/chi_xact_field_eb.hpp
+++ b/chi/xact/chi_xact_field_eb.hpp
@@ -519,7 +519,7 @@ namespace CHI {
             inline constexpr SnoopFieldMappingBack SnpStashShared                   (Y , Y , Y , Y , Y , Y , I0, S , Y , Y , S , A1, I0, Y , Y );
-            inline constexpr SnoopFieldMappingBack SnpQuery                         (Y , Y , Y , Y , Y , Y , I0, I0, I0, I0, I0, A1, I0, Y , D );
+            inline constexpr SnoopFieldMappingBack SnpQuery                         (Y , Y , Y , Y , Y , Y , I0, I0, I0, I0, I0, I0, I0, Y , D );
             inline constexpr SnoopFieldMappingBack SnpDVMOp                         (Y , Y , Y , Y , Y , I0, I0, S , S , S , Y , I0, I0, Y , I0);
```
