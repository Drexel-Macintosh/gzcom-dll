# Verification report: flora-static-data, vtable-order, signatures

Independent check of three header branches against their common base `aab858d`
(= `upstream/master`), using clang instead of MSVC. Nothing on the three branches
was changed.

| Branch | Head | Commits |
|---|---|---|
| `flora-static-data` | `1c9f31d` | 1 |
| `vtable-order` | `ddfddb6` | 14 |
| `signatures` | `15bd60c` | 21 |

**Result: no UNEXPECTED row.** Every vtable slot that differs is one a commit message names
(directly, or as "the pair/group compiled swapped"); every other slot is byte-for-byte
the same signature at the same index; slot counts change only for the three classes
whose commit messages say so; every argument-size change in `signatures` is the one
its commit message states; the `cSC4StaticFloraData` layout is identical. The branches
merge with each other cleanly, and the merged tree parses exactly as `aab858d` does.

One wording point in a commit message (not in the code) is noted under `vtable-order`.

## Method

- Compiler: Ubuntu clang 18.1.3. Flags on every compile:
  `--target=i686-pc-windows-msvc -fms-extensions -fms-compatibility -std=c++17`.
- There is no MSVC SDK in the container, so standard headers come from libstdc++ 13
  (`-isystem /usr/include/c++/13 ...`), with an empty `gnu/stubs-32.h` stub and
  `-D__GCC_ATOMIC_TEST_AND_SET_TRUEVAL=1` (needed by `<atomic>` under the MSVC target).
  Neither touches any SDK type.
- **Vtables.** `-Xclang -fdump-vtable-layouts` prints nothing under `-fsyntax-only`
  (clang computes vftables during code generation), so each probe was compiled with
  `-S -emit-llvm -o /dev/null` instead. The probe TU is
  `#include "<Class>.h"` + `struct Probe__ : <Class> { Probe__(); }; Probe__::Probe__() {}`,
  which forces the complete vftable of the class (including inherited slots) to be laid
  out. clang's Microsoft ABI applies MSVC's rule of placing virtual overloads of one name
  at the first one's slot in reverse declaration order, so the dump is the slot order
  MSVC produces. The class's vftable was dumped on `aab858d` and on the branch, and diffed
  line by line. Slots below are 0-based vftable indices (`QueryInterface` = 0), i.e. the
  dump's index minus the RTTI entry.
- **Argument stack size.** For each changed method, its parameter list was put on a
  definition `extern "C" void __stdcall probe(params...)`; clang's MSVC decoration
  `_probe@N` gives N = bytes the callee pops (each parameter rounded up to 4). With
  `__thiscall` `this` is in ECX, so N is the method's stack size. A class-type return
  value adds a 4-byte hidden return pointer; the only such case (`OverlayColor`) was
  also checked in the IR of a call through the interface:
  `call x86_thiscallcc void %8(ptr this, ptr sret(%class.cS3DVector3), i32 7)` at
  vftable index 24 (hidden pointer first, then the id).
- **Record layout.** `-fsyntax-only -Xclang -fdump-record-layouts` on a TU that takes
  `sizeof(cSC4StaticFloraData)`.
- **Merge.** `git merge-tree --write-tree` for each pair, and a real merge of all three
  (`flora-static-data` + `vtable-order` + `signatures`); then `-fsyntax-only` over every
  `gzcom-dll/src/*.cpp` and over every `gzcom-dll/include/*.h` included alone, on
  `aab858d` and on the merged tree.

## vtable-order

Every changed header compiles without error on both sides. Rows are all slots that
differ between `aab858d` and the branch.

| Class | Slot | Before | After | Verdict |
|---|---|---|---|---|
| cIGZCanvas | 3 | `bool Init(const SC4Rect<int32_t> &)` | `bool Init(uint32_t, uint32_t, uint32_t)` | intended |
| cIGZCanvas | 4 | `bool Init(uint32_t, uint32_t, uint32_t)` | `bool Init(const cRZRect &)` | intended |
| cIGZFontSys | 13 | `bool AddFont(const cIGZString &)` | `bool AddFont(cIGZFont *)` | intended |
| cIGZFontSys | 14 | `bool AddFont(cIGZFont *)` | `bool AddFont(const cIGZString &)` | intended |
| cIGZWinCtrlMgr | 9 | `cIGZWinBtn *CreateButton(uint32_t, uint32_t, const cIGZString &, cIGZBuffer *, const cRZRect *, uint32_t)` | `cIGZWinBtn *CreateButton(uint32_t, const cIGZString &, cIGZBuffer *, const cRZRect *, uint32_t)` | intended |
| cIGZWinCtrlMgr | 10 | `cIGZWinBtn *CreateButton(uint32_t, const cIGZString &, cIGZBuffer *, const cRZRect *, uint32_t)` | `cIGZWinBtn *CreateButton(uint32_t, uint32_t, const cIGZString &, cIGZBuffer *, const cRZRect *, uint32_t)` | intended |
| cIGZWinCtrlMgr | 31 | `uint32_t DefaultColor(int32_t)` | `void DefaultColor(int32_t, uint8_t &, uint8_t &, uint8_t &)` | intended |
| cIGZWinCtrlMgr | 32 | `void DefaultColor(int32_t, uint8_t &, uint8_t &, uint8_t &)` | `uint32_t DefaultColor(int32_t)` | intended |
| cIGZWinCtrlMgr | 33 | `void SetDefaultColor(int32_t, uint32_t)` | `void SetDefaultColor(int32_t, uint8_t, uint8_t, uint8_t)` | intended |
| cIGZWinCtrlMgr | 34 | `void SetDefaultColor(int32_t, uint8_t, uint8_t, uint8_t)` | `void SetDefaultColor(int32_t, uint32_t)` | intended |
| cIGZWinTextEdit | 12 | `bool SetFontColor(uint32_t)` | `bool SetFontColor(ColorIndex, uint32_t)` | intended |
| cIGZWinTextEdit | 13 | `bool SetFontColor(ColorIndex, uint32_t)` | `bool SetFontColor(uint32_t)` | intended |
| cIGZWinTextEdit | 44 | `const char *GetText()` | `void GetText(char *, uint32_t *)` | intended |
| cIGZWinTextEdit | 46 | `void GetText(char *, uint32_t *)` | `const char *GetText()` | intended |
| cIGZWinTextEdit | 48 | `bool SetText(char *, uint32_t, bool)` | `void SetText(const cIGZString &, bool)` | intended |
| cIGZWinTextEdit | 49 | `void SetText(const cIGZString &, bool)` | `bool SetText(char *, uint32_t, bool)` | intended |
| cIGZWinTextEdit | 50 | `void InsertText(char *, uint32_t, uint32_t)` | `void InsertText(char *, uint32_t, uint32_t, uint32_t)` | intended |
| cIGZWinTextEdit | 51 | `void InsertText(char *, uint32_t, uint32_t, uint32_t)` | `void InsertText(char *, uint32_t, uint32_t)` | intended |
| cIGZWinTextEdit | 53 | `uint32_t RemoveText(uint32_t, uint32_t, cIGZString *)` | `uint32_t RemoveText(uint32_t, uint32_t, uint32_t, uint32_t, cIGZString *)` | intended |
| cIGZWinTextEdit | 55 | `uint32_t RemoveText(uint32_t, uint32_t, uint32_t, uint32_t, cIGZString *)` | `uint32_t RemoveText(uint32_t, uint32_t, cIGZString *)` | intended |
| cISC43DRender | 22 | `bool TakeSnapshot(cIGZBuffer **, bool)` | `bool TakeSnapshot(cIGZString &, bool)` | intended |
| cISC43DRender | 23 | `bool TakeSnapshot(cIGZString &, bool)` | `bool TakeSnapshot(cIGZBuffer **, bool)` | intended |
| cISC43DRender | 42 | `bool StartScreenShake(float, float)` | `bool StartScreenShake(const cSC4ShakeDescription &, const cS3DVector2 *)` | intended |
| cISC43DRender | 43 | `bool StartScreenShake(const cSC4ShakeDescription &, const cS3DVector2 *)` | `bool StartScreenShake(float, float)` | intended |
| cISC43DRender | 53 | `bool SetBackgroundPicture(cIGZBuffer *, bool)` | `bool SetBackgroundPicture(uint32_t, int32_t, int32_t, bool)` | intended |
| cISC43DRender | 54 | `bool SetBackgroundPicture(uint32_t, int32_t, int32_t, bool)` | `bool SetBackgroundPicture(cIGZBuffer *, bool)` | intended |
| cISC43DRender | 60 | `bool PickTerrain(const float *, const float *, float *, bool, bool)` | `bool PickTerrain(int32_t, int32_t, float *, bool, bool)` | intended |
| cISC43DRender | 61 | `bool PickTerrain(int32_t, int32_t, float *, bool, bool)` | `bool PickTerrain(const float *, const float *, float *, bool, bool)` | intended |
| cISC43DRender | 62 | `bool PickTerrainAlways(const float *, const float *, float *, bool)` | `bool PickTerrainAlways(int32_t, int32_t, float *, bool)` | intended |
| cISC43DRender | 63 | `bool PickTerrainAlways(int32_t, int32_t, float *, bool)` | `bool PickTerrainAlways(const float *, const float *, float *, bool)` | intended |
| cISC43DRender | 64 | `bool Pick(int32_t, int32_t, cISC4OccupantFilter *, cIS3DModelInstance *&)` | `bool Pick(int32_t, int32_t, const float *, const float *, cISC4OccupantFilter *, cIS3DModelInstance *&)` | intended |
| cISC43DRender | 65 | `bool Pick(int32_t, int32_t, const float *, const float *, cISC4OccupantFilter *, cIS3DModelInstance *&)` | `bool Pick(int32_t, int32_t, cISC4OccupantFilter *, cIS3DModelInstance *&)` | intended |
| cISC4ConstructionOccupant | 5 | `bool Init()` | (none) | intended |
| cISC4ConstructionOccupant | 6 | `bool Shutdown()` | (none) | intended |
| cISC4ConstructionOccupant | 7 | `bool Write(cIGZOStream &)` | (none) | intended |
| cISC4ConstructionOccupant | 8 | `bool Read(cIGZIStream &)` | (none) | intended |
| cISC4ConstructionOccupant | 9 | `uint32_t GetGZCLSID()` | (none) | intended |
| cISC4ConstructionOccupant | 10 | `bool GetPosition(float *)` | (none) | intended |
| cISC4ConstructionOccupant | 11 | `void SetPosition(const float *)` | (none) | intended |
| cISC4ConstructionOccupant | 12 | `bool GetBoundingBox(float *, float *)` | (none) | intended |
| cISC4ConstructionOccupant | 13 | `cIGZUnknown *GetOwner()` | (none) | intended |
| cISC4ConstructionOccupant | 14 | `void SetOwner(cIGZUnknown *)` | (none) | intended |
| cISC4ConstructionOccupant | 15 | `bool GetBounds(cS3DBoundingBox &)` | (none) | intended |
| cISC4ConstructionOccupant | 16 | `bool GetWorldBounds(cS3DBoundingBox &)` | (none) | intended |
| cISC4ConstructionOccupant | 17 | `bool GetTransform(cS3DTransform &)` | (none) | intended |
| cISC4ConstructionOccupant | 18 | `bool SetTransform(const cS3DTransform &)` | (none) | intended |
| cISC4ConstructionOccupant | 19 | `bool GetSortValue(uint32_t &)` | (none) | intended |
| cISC4ConstructionOccupant | 20 | `bool Draw(SC4DrawContext *, bool)` | (none) | intended |
| cISC4ConstructionOccupant | 21 | `bool DoMessage(cIGZMessage2 *)` | (none) | intended |
| cISC4LightingManager | 19 | `cS3DVector3 GetColor() const` | `void GetColor(const cS3DVector3 &, const cS3DVector3 &, unsigned char &, unsigned char &, unsigned char &) const` | intended |
| cISC4LightingManager | 20 | `cS3DVector3 GetColor(const cS3DVector3 &) const` | `void GetColor(const cS3DVector3 &, unsigned char &, unsigned char &, unsigned char &) const` | intended |
| cISC4LightingManager | 21 | `cS3DVector3 GetColor(const cS3DVector3 &, const cS3DVector3 &) const` | `void GetColor(unsigned char &, unsigned char &, unsigned char &) const` | intended |
| cISC4LightingManager | 22 | `void GetColor(unsigned char &, unsigned char &, unsigned char &) const` | `cS3DVector3 GetColor(const cS3DVector3 &, const cS3DVector3 &) const` | intended |
| cISC4LightingManager | 23 | `void GetColor(const cS3DVector3 &, unsigned char &, unsigned char &, unsigned char &) const` | `cS3DVector3 GetColor(const cS3DVector3 &) const` | intended |
| cISC4LightingManager | 24 | `void GetColor(const cS3DVector3 &, const cS3DVector3 &, unsigned char &, unsigned char &, unsigned char &) const` | `cS3DVector3 GetColor() const` | intended |
| cISC4NetworkManager | 13 | `bool GetAllIntersectionPieces2(cISC4NetworkOccupant *, SC4List<cISC4Occupant *> &, cISC4OccupantManager *, cISC4NetworkOccupant *, cISC4OccupantFilter *)` | `uint32_t MapPreviewTexture(uint32_t, uint8_t &)` | intended |
| cISC4NetworkManager | 14 | `uint32_t MapPreviewTexture(uint32_t, uint8_t &)` | `uint32_t MapShadowTexture(uint32_t, uint8_t &, uint32_t, uint32_t)` | intended |
| cISC4NetworkManager | 15 | `uint32_t MapShadowTexture(uint32_t, uint8_t &, uint32_t, uint32_t)` | `uint32_t MapPath(uint32_t)` | intended |
| cISC4NetworkManager | 16 | `uint32_t MapPath(uint32_t)` | `bool RegisterConstructionCrew(intptr_t)` | intended |
| cISC4NetworkManager | 17 | `bool RegisterConstructionCrew(intptr_t)` | `bool UnregisterConstructionCrew(intptr_t)` | intended |
| cISC4NetworkManager | 18 | `bool UnregisterConstructionCrew(intptr_t)` | `cISC4OccupantFilter *GetWaterOccupantFilter()` | intended |
| cISC4NetworkManager | 19 | `cISC4OccupantFilter *GetWaterOccupantFilter()` | `cISC4OccupantFilter *GetSubwayOccupantFilter()` | intended |
| cISC4NetworkManager | 20 | `cISC4OccupantFilter *GetSubwayOccupantFilter()` | `bool CreateConnectionNode(cISC4Occupant *&, int32_t, const SC4Point<int> &, uint32_t, uint32_t, uint32_t, uint8_t)` | intended |
| cISC4NetworkManager | 21 | `bool CreateConnectionNode(cISC4Occupant *&, int32_t, const SC4Point<int> &, uint32_t, uint32_t, uint32_t, uint8_t)` | `bool AutoconnectLotNodes(cISC4Lot *, const SC4List<cISC4Occupant *> &)` | intended |
| cISC4NetworkManager | 22 | `bool AutoconnectLotNodes(cISC4Lot *, const SC4List<cISC4Occupant *> &)` | `const char *GetEnglishNetworkName(int32_t)` | intended |
| cISC4NetworkManager | 23 | `const char *GetEnglishNetworkName(int32_t)` | `bool AddToRetrofitList(cISC4NetworkOccupant *)` | intended |
| cISC4NetworkManager | 24 | `bool AddToRetrofitList(cISC4NetworkOccupant *)` | (none) | intended |
| cISC4NetworkTool | 16 | `void ClearOccupantHighlights()` | `void ClearCellPreviews()` | intended |
| cISC4NetworkTool | 17 | `void ClearCellPreviews()` | `void ClearOccupantHighlights()` | intended |
| cISC4PropManager | 21 | `bool RemoveCityProps(const SC4Rect<int32_t> &)` | `bool RemoveCityProps(const SC4Rect<float> &)` | intended |
| cISC4PropManager | 22 | `bool RemoveCityProps(const SC4Rect<float> &)` | `bool RemoveCityProps(const SC4Rect<int32_t> &)` | intended |
| cISC4PylonOccupant | 3 | `bool Init()` | `cISC4Occupant *AsOccupant()` | intended |
| cISC4PylonOccupant | 4 | `bool Shutdown()` | `bool Init()` | intended |
| cISC4PylonOccupant | 5 | `void SetUIState(tUIState)` | `bool Shutdown()` | intended |
| cISC4PylonOccupant | 6 | `tUIState GetUIState()` | `void SetUIState(tUIState)` | intended |
| cISC4PylonOccupant | 7 | `void SetBitmapInfo(uint32_t, const cIGZString &)` | `tUIState GetUIState()` | intended |
| cISC4PylonOccupant | 8 | `void SetDispatchType(cISC4DispatchManager::tDispatchType)` | `void SetBitmapInfo(uint32_t, const cIGZString &)` | intended |
| cISC4PylonOccupant | 9 | `cISC4DispatchManager::tDispatchType GetDispatchType()` | `void SetDispatchType(cISC4DispatchManager::tDispatchType)` | intended |
| cISC4PylonOccupant | 10 | `void SetDispatchState(cISC4DispatchManager::tDispatchState)` | `cISC4DispatchManager::tDispatchType GetDispatchType()` | intended |
| cISC4PylonOccupant | 11 | `cISC4DispatchManager::tDispatchState GetDispatchState()` | `void SetDispatchState(cISC4DispatchManager::tDispatchState)` | intended |
| cISC4PylonOccupant | 12 | `void SetCallNumber(uint16_t)` | `cISC4DispatchManager::tDispatchState GetDispatchState()` | intended |
| cISC4PylonOccupant | 13 | `uint16_t GetCallNumber()` | `void SetCallNumber(uint16_t)` | intended |
| cISC4PylonOccupant | 14 | `void SetDragState(bool)` | `uint16_t GetCallNumber()` | intended |
| cISC4PylonOccupant | 15 | `bool GetDragState()` | `void SetDragState(bool)` | intended |
| cISC4PylonOccupant | 16 | `void SetSourceOccupant(cISC4Occupant *)` | `bool GetDragState()` | intended |
| cISC4PylonOccupant | 17 | `cISC4Occupant *GetSourceOccupant()` | `void SetSourceOccupant(cISC4Occupant *)` | intended |
| cISC4PylonOccupant | 18 | `void SetPylonOwner(cIGZUnknown *)` | `cISC4Occupant *GetSourceOccupant()` | intended |
| cISC4PylonOccupant | 19 | `cIGZUnknown *GetPylonOwner()` | `void SetPylonOwner(cIGZUnknown *)` | intended |
| cISC4PylonOccupant | 20 | (none) | `cIGZUnknown *GetPylonOwner()` | intended |
| cISC4RegionalCity | 47 | `void GetWorldBaseElevation(float)` | `float GetWorldBaseElevation() const` | intended |
| cISC4RegionalCity | 48 | `float GetWorldBaseElevation() const` | `void SetWorldBaseElevation(float)` | intended |
| cISLWinLocationSaver | 5 | `bool SaveWindowLocation(cIGZWin *)` | `bool SaveWindowLocation(uint32_t, int32_t, int32_t)` | intended |
| cISLWinLocationSaver | 6 | `bool SaveWindowLocation(uint32_t, int32_t, int32_t)` | `bool SaveWindowLocation(cIGZWin *)` | intended |
| cISLWinLocationSaver | 7 | `bool GetWindowRestoreLocation(cIGZWin *, int32_t &, int32_t &, bool)` | `bool GetWindowRestoreLocation(uint32_t, int32_t &, int32_t &, bool)` | intended |
| cISLWinLocationSaver | 8 | `bool GetWindowRestoreLocation(uint32_t &, int32_t &, int32_t &, bool)` | `bool GetWindowRestoreLocation(cIGZWin *, int32_t &, int32_t &, bool)` | intended |
| cISLWinLocationSaver | 9 | `bool RestoreWindowLocation(cIGZWin *, bool)` | `bool RestoreWindowLocation(uint32_t, bool)` | intended |
| cISLWinLocationSaver | 10 | `bool RestoreWindowLocation(uint32_t, bool)` | `bool RestoreWindowLocation(cIGZWin *, bool)` | intended |
| cISTESurfaceWater | 7 | `bool IsNavigable(int32_t, int32_t, int32_t, cISC4Lot *) const` | `bool IsNavigable(cS3DVector3, int32_t, cISC4Lot *) const` | intended |
| cISTESurfaceWater | 8 | `bool IsNavigable(cS3DVector3 *, int32_t, cISC4Lot *) const` | `bool IsNavigable(int32_t, int32_t, int32_t, cISC4Lot *) const` | intended |

| Class | Slots before | Slots after | Changed slots |
|---|---|---|---|
| cIGZCanvas | 21 | 21 | 2 |
| cIGZFontSys | 24 | 24 | 2 |
| cIGZWinCtrlMgr | 43 | 43 | 6 |
| cIGZWinTextEdit | 90 | 90 | 10 |
| cISC43DRender | 75 | 75 | 12 |
| cISC4CitySituationAdvice | 56 | 56 | 0 |
| cISC4ConstructionOccupant | 22 | 5 | 17 |
| cISC4LightingManager | 29 | 29 | 6 |
| cISC4NetworkManager | 25 | 24 | 12 |
| cISC4NetworkTool | 51 | 51 | 2 |
| cISC4PropManager | 40 | 40 | 2 |
| cISC4PylonOccupant | 20 | 21 | 18 |
| cISC4RegionalCity | 73 | 73 | 2 |
| cISLWinLocationSaver | 11 | 11 | 6 |
| cISTESurfaceWater | 14 | 14 | 2 |

Notes:

- Slot counts change only for `cISC4ConstructionOccupant` (22 → 5: "keep only the game's
  two methods", 3 + `AsOccupant` + `SetLot`), `cISC4NetworkManager` (25 → 24:
  `GetAllIntersectionPieces2` removed at slot 13, everything after it moves up one) and
  `cISC4PylonOccupant` (20 → 21: `AsOccupant` inserted at slot 3, everything after it
  moves down one). All three are what the commit messages state.
- `cIGZWinTextEdit`: slots 45 (`GetText(cIGZString &)`) and 54
  (`RemoveText(uint32_t, uint32_t, uint32_t, cIGZString *)`) do not appear because they
  are the middle member of three-overload groups and stay in place when the group is
  reversed.
- Resulting slots match the slot numbers given in the commit messages: `cIGZCanvas`
  `Init(uint32_t, uint32_t, uint32_t)` 3 / `Init(const cRZRect &)` 4; `cIGZFontSys`
  `AddFont(cIGZFont *)` 13 / `AddFont(const cIGZString &)` 14; `cISC4PropManager`
  `RemoveCityProps(SC4Rect<float>)` 21 / `(SC4Rect<int32_t>)` 22; `cISTESurfaceWater`
  `IsNavigable(cS3DVector3, ...)` 7 / cell version 8; `cISC4NetworkTool`
  `ClearCellPreviews` 16 / `ClearOccupantHighlights` 17; `cISC4RegionalCity`
  `GetWorldBaseElevation` 47 / `SetWorldBaseElevation` 48; `cISC4LightingManager`
  `GetColor` ×6 at 19–24; `cISC4NetworkManager` `MapPreviewTexture` 13 (slot 12
  `GetAllIntersectionPieces` unchanged); `cISC4PylonOccupant` `AsOccupant` 3.
- Argument sizes the commit messages quote, measured as for `signatures`:
  `cISTESurfaceWater::IsNavigable(cS3DVector3, int32_t, cISC4Lot *) const` = 20 B
  (was `cS3DVector3 *` = 12 B), matching "pops 20 bytes"; `cIGZCanvas` `Init` pair
  12 B vs 4 B, a difference of 8 B, matching "the stack was left 8 bytes off";
  `cISLWinLocationSaver::GetWindowRestoreLocation(uint32_t, ...)` 16 B before and after
  (`uint32_t &` → `uint32_t`).
- `cISC4CitySituationAdvice` (`const const` → `const`): no slot or signature change, as
  the commit says. Wording point: the commit message says clang and GCC reject the
  duplicated `const`. GCC 13.3 does reject it (`error: duplicate cv-qualifier`); clang
  18.1.3 only warns (`warning: duplicate 'const' declaration specifier`
  [-Wduplicate-decl-specifier]) for both the MSVC and Linux targets. The code change is
  correct either way.

## signatures

Every changed header compiles without error on both sides. "args" = bytes the callee
pops under MSVC `__thiscall` (parameters rounded to 4, plus 4 for a hidden return pointer).

| Class | Slot | Before (args) | After (args) | Verdict |
|---|---|---|---|---|
| cIGZFrameWork | 16 | `bool OnIdle(uint32_t)` (4 B) | `bool OnIdle()` (0 B) | intended |
| cIGZLanguageManager | 45 | `bool ConvertMoneyFromLocaleToLocale(uint32_t, int32_t *, uint32_t)` (12 B) | `bool ConvertMoneyFromLocaleToLocale(int32_t, uint32_t, int32_t &, uint32_t)` (16 B) | intended |
| cIGZPersistResourceManager | 22 | `bool FindObjectFactory(cIGZPersistResourceFactory **)` (4 B) | `bool FindObjectFactory(const cGZPersistResourceKey &, cIGZPersistResourceFactory **)` (8 B) | intended |
| cIGZSndSys | 5 | `bool DoEvent(cIGZString *, uint32_t, uint32_t, uint32_t, uint32_t)` (20 B) | `bool DoEvent(uint32_t, cIGZString *, uint32_t, uint32_t, uint32_t, uint32_t)` (24 B) | intended |
| cIGZSndSys | 9 | `bool PostEvent(cIGZString *, uint32_t, uint32_t, uint32_t, uint32_t)` (20 B) | `bool PostEvent(uint32_t, cIGZString *, uint32_t, uint32_t, uint32_t, uint32_t)` (24 B) | intended |
| cIGZWinGen | 31 | `bool RemoveColorBar()` (0 B) | `bool RemoveColorBar(int32_t)` (4 B) | intended |
| cISC4Automaton | 65 | `bool HandleCollision(const cS3DVector3 &)` (4 B) | `bool HandleCollision(cISC4Automaton *, const cS3DVector3 &)` (8 B) | intended |
| cISC4BuildingDevelopmentSimulator | 9 | `int64_t GetCostOfOccupantDemolition()` (0 B) | `int64_t GetCostOfOccupantDemolition(cISC4FoundationOccupant *, uint32_t)` (8 B) | intended |
| cISC4DBSegmentIStream | 21 | `bool Open(cISC4DBSegment *, const cGZPersistResourceKey &, bool)` (12 B) | `bool Open(cISC4DBSegment *, const cGZPersistResourceKey &)` (8 B) | intended |
| cISC4DBSegmentIStream | 26 | `bool ReadGZSerializable(cIGZSerializable **)` (4 B) | `bool ReadGZSerializable(cIGZSerializable **, bool)` (8 B) | intended |
| cISC4DisasterInstanceVolcano | 5 | `bool SetLocation(int32_t, int32_t)` (8 B) | `bool SetLocation(const cRZPoint &)` (4 B) | intended |
| cISC4DisasterLayer | 9 | `bool DisasterHasEnded(cISC4DisasterInstance *)` (4 B) | `bool DisasterHasEnded()` (0 B) | intended |
| cISC4DispatchManager | 9 | `bool RevokeAllDispatches()` (0 B) | `bool RevokeAllDispatches(tDispatchType)` (4 B) | intended |
| cISC4HistoryWarehouse | 9 | `bool Coalesce(uint32_t, const cIGZDate &, int32_t)` (12 B) | `bool Coalesce(uint32_t, const cIGZDate &, const cIGZDate &, int32_t)` (16 B) | intended |
| cISC4LotConfigurationManager | 5 | `bool AddLotConfiguration(const cGZPersistResourceKey &, void *)` (8 B) | `bool AddLotConfiguration(const cGZPersistResourceKey &)` (4 B) | intended |
| cISC4LotDeveloper | 53 | `bool RemoveLotFoundation(cISC4Lot *)` (4 B) | `bool RemoveLotFoundation(int32_t, int32_t)` (8 B) | intended |
| cISC4NetworkTool | 10 | `void UpdateOnZoneChange()` (0 B) | `void UpdateOnZoneChange(SC4Rect<int32_t>, uint32_t)` (20 B) | intended |
| cISC4PoliceSimulator | 37 | `int64_t GetJailCurrentUpkeepCost()` (0 B) | `int64_t GetJailCurrentUpkeepCost(cISC4Occupant *)` (4 B) | intended |
| cISC4PoliceSimulator | 39 | `bool IsInJailbreak()` (0 B) | `bool IsInJailbreak(cISC4Occupant *)` (4 B) | intended |
| cISC4ResidentialSimulator | 56 | `bool GetAutoBudget()` (0 B) | `bool GetAutoBudget(cISC4Occupant *) const` (4 B) | intended |
| cISC4ResidentialSimulator | 57 | `bool SetAutoBudget(bool)` (4 B) | `bool SetAutoBudget(cISC4Occupant *, bool)` (8 B) | intended |
| cISC4SeaportDeveloper | 11 | `uint32_t GetNumSeaports(ePortSize)` (4 B) | `uint32_t GetNumSeaports()` (0 B) | intended |
| cISC4TutorialSystem | 13 | `bool FilterMessage(uint32_t)` (4 B) | `bool FilterMessage(cGZMessage &)` (4 B) | intended |
| cISCPropertyHolder | 11 | `bool AddProperty(uint32_t, int32_t, bool)` (12 B) | `bool AddProperty(uint32_t, cIGZUnknown *, bool)` (12 B) | intended |
| cISCPropertyHolder | 12 | `bool AddProperty(uint32_t, const cIGZString &)` (8 B) | `bool AddProperty(uint32_t, const cIGZString &, bool)` (12 B) | intended |
| cISCPropertyHolder | 17 | `bool RemoveProperty(uint32_t)` (4 B) | `bool RemoveProperty(uint32_t, bool)` (8 B) | intended |
| cISCPropertyHolder | 18 | `bool RemoveAllProperties()` (0 B) | `bool RemoveAllProperties(bool)` (4 B) | intended |
| cISCPropertyHolder | 19 | `bool EnumProperties(FunctionPtr2, FunctionPtr1) const` (8 B) | `bool EnumProperties(FunctionPtr2, FunctionPtr1, void *) const` (12 B) | intended |
| cISTEOverlayManager | 24 | `bool OverlayColor(uint32_t, cS3DVector3 &) const` (8 B) | `cS3DVector3 OverlayColor(uint32_t) const` (8 B incl. 4 B hidden return ptr) | intended |

| Class | Slots before | Slots after | Changed slots |
|---|---|---|---|
| cIGZFrameWork | 40 | 40 | 1 |
| cIGZLanguageManager | 46 | 46 | 1 |
| cIGZPersistResourceManager | 44 | 44 | 1 |
| cIGZSndSys | 51 | 51 | 2 |
| cIGZWinGen | 32 | 32 | 1 |
| cISC4Automaton | 70 | 70 | 1 |
| cISC4BuildingDevelopmentSimulator | 59 | 59 | 1 |
| cISC4DBSegmentIStream | 29 | 29 | 2 |
| cISC4DisasterInstanceVolcano | 9 | 9 | 1 |
| cISC4DisasterLayer | 19 | 19 | 1 |
| cISC4DispatchManager | 23 | 23 | 1 |
| cISC4HistoryWarehouse | 14 | 14 | 1 |
| cISC4LotConfigurationManager | 28 | 28 | 1 |
| cISC4LotDeveloper | 65 | 65 | 1 |
| cISC4NetworkTool | 51 | 51 | 1 |
| cISC4PoliceSimulator | 45 | 45 | 2 |
| cISC4ResidentialSimulator | 60 | 60 | 2 |
| cISC4SeaportDeveloper | 17 | 17 | 1 |
| cISC4TutorialSystem | 18 | 18 | 1 |
| cISCPropertyHolder | 22 | 22 | 5 |
| cISTEOverlayManager | 28 | 28 | 1 |

Every before/after byte count above is the one stated (or implied) by its commit message;
no slot moved and no slot count changed. Changes beyond the argument list, all stated in
the commit messages:

- `cISTEOverlayManager::OverlayColor` returns `cS3DVector3` by value: 8 B before and
  after, but the hidden return pointer is now first and the id second, as the commit says.
- `cISC4ResidentialSimulator::GetAutoBudget` also becomes `const`; the commit message
  gives the signature with `const`. This changes no slot and no stack size.
- `cISC4LotConfigurationManager::AddLotConfiguration(key, void*)` and
  `cISC4DBSegmentIStream::Open(segment, key, bool)` are kept as non-virtual forwarders
  (not in the vftable; the dumps confirm the slot counts are unchanged).
- `cISC4TutorialSystem::FilterMessage` and `cISCPropertyHolder::AddProperty` slot 11
  change parameter type at the same size (4 B pointer/reference for 4 B integer).

## flora-static-data

`-fdump-record-layouts` of `cSC4StaticFloraData`, before (`aab858d`) and after (`1c9f31d`).
The two dumps differ only in the names at 0x0 and 0x8; offsets and sizes are identical.
The new `tParameters` holds only an enum and adds no data member. The branch's new
`static_assert`s (`type` at 0x0, `seedType` at 0x8) and the existing ones pass.

| Offset | Before | After | Verdict |
|---|---|---|---|
| 0x0 | `uint32_t unknown1` | `uint32_t type` | intended (rename) |
| 0x4 | `uint32_t clusterType` | `uint32_t clusterType` | unchanged (comment only) |
| 0x8 | `uint32_t type` | `uint32_t seedType` | intended (rename) |
| 0xC | `bool bWaterFlora` | same | unchanged |
| 0x10 | `float parameters[14]` | same | unchanged |
| 0x48 | `float clusterMaxRadius` | same | unchanged |
| 0x4C | `float clusterMaxHeight` | same | unchanged |
| 0x50 | `FloraPreferencesEntry floraPreferences[16]` (16 B each) | same | unchanged |
| 0x150 | `uint32_t cursorID` | same | unchanged |
| 0x154 | `SC4Vector<uint32_t> families` (12 B) | same | unchanged |
| 0x160 | `int64_t plopCost` | same | unchanged |
| 0x168 | `cRZAutoRefCount<cISCResExemplar> exemplar` | same | unchanged |
| 0x16C | `uint8_t flags` | same | unchanged (comment only) |
| 0x170 | `uint32_t refCount` | same | unchanged |
| size | 376 (0x178), align 8 | 376 (0x178), align 8 | identical |

Full layout from the branch:

```
0 | struct cSC4StaticFloraData::FloraPreferencesEntry
         0 |   uint8_t[16] values
           | [sizeof=16, align=1,
           |  nvsize=16, nvalign=1]

         0 | class cSC4StaticFloraData
         0 |   uint32_t type
         4 |   uint32_t clusterType
         8 |   uint32_t seedType
        12 |   _Bool bWaterFlora
        16 |   float[14] parameters
        72 |   float clusterMaxRadius
        76 |   float clusterMaxHeight
        80 |   FloraPreferencesEntry[16] floraPreferences
       336 |   uint32_t cursorID
       340 |   class SC4Vector<unsigned int> families
       340 |     unsigned int * mpStart
       344 |     unsigned int * mpEnd
       348 |     unsigned int * mpCapacity
       352 |   int64_t plopCost
       360 |   class cRZAutoRefCount<class cISCResExemplar> exemplar
       360 |     class cISCResExemplar * pObject
       364 |   uint8_t flags
       368 |   uint32_t refCount
           | [sizeof=376, align=8,
           |  nvsize=376, nvalign=8]

         0 | struct cSC4StaticFloraData::tFloraFlags (empty)
           | [sizeof=1, align=1,
           |  nvsize=0, nvalign=1]
```

Existing source that names `cSC4StaticFloraData::type` still compiles but now reads the
field at 0x0 instead of 0x8, as the commit message states. Nothing in `gzcom-dll/src`
uses `cSC4StaticFloraData`.

## Merging the three branches

| Merge | Result |
|---|---|
| `flora-static-data` + `vtable-order` | clean |
| `flora-static-data` + `signatures` | clean |
| `vtable-order` + `signatures` | clean |
| all three | clean |

The only file touched by two branches is `cISC4NetworkTool.h` (`vtable-order` and
`signatures`). On the merged tree its vftable has both changes (slot 10
`UpdateOnZoneChange(SC4Rect<int32_t>, uint32_t)`, slots 16/17 `ClearCellPreviews` /
`ClearOccupantHighlights`) and is otherwise identical. Every other changed class has
exactly the vftable of its own branch on the merged tree.

### src/ -fsyntax-only

All 31 `gzcom-dll/src/*.cpp` files were parsed with the flags above on `aab858d`, on each
branch and on the merged tree. The same 27 files pass everywhere; the same 4 fail
everywhere with the same error counts and first errors. All four errors come from
standard-library names that MSVC's headers include transitively and libstdc++'s do not
(`std::exchange`, `memcpy`, `memset`, `NULL`); none involves a header the branches change.

| File | Errors on aab858d | Errors on merged | First error (same on both) |
|---|---|---|---|
| SC4String.cpp | 13 | 13 | `no member named 'exchange' in namespace 'std'; did you mean '__exchange'?` |
| cRZBaseVariant.cpp | 20 | 20 | `use of undeclared identifier 'memcpy'; did you mean 'wmemcpy'?` |
| cRZMessage2.cpp | 1 | 1 | `use of undeclared identifier 'NULL'` |
| cRZMessage2Standard.cpp | 4 | 4 | `use of undeclared identifier 'memset'; did you mean 'wmemset'?` |

### include/ -fsyntax-only (each header alone)

All 292 headers in `gzcom-dll/include` were parsed alone, on `aab858d` and on the merged
tree: identical results. 286 pass. The same 6 fail on both, none of them changed by the
branches: `GZWinKeyAcceleratorUtil.h`, `cIGZFrameWorkW32.h`, `cIGZWinProcFilterW32.h`
(`Windows.h` not available), `cIGZOStream.h`, `cISC4DBSegmentOStream.h`
(`unknown type name 'cIGZVariant'` in `cIGZOStream.h`) and `SC4NewCitySpecification.h`
(`'SC4NewCitySpecification' does not refer to a value`).
