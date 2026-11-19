# Loadout assembly — the parts the MCP surface cannot create

`create_spec` and `create_item_set` build two of the four loadout families. The other two, and jewel sockets, need a small XML edit. Do it while PoB holds the build **only in this order**: save via `lua_save_build`, edit the file, then `lua_reload_build`. Editing before saving loses the in-memory work.

PoB writes UTF-8 without BOM; write back the same way, or PoB loads a blank level-1 build:

```powershell
[System.IO.File]::WriteAllText($path, $xml, (New-Object System.Text.UTF8Encoding($false)))
```

## 1. Jewel sockets live on the tree spec, not in the item set

```xml
<Tree activeSpec="4">
  <Spec … title="BUDGET">
    <URL>…</URL>
    <Sockets>
      <Socket itemId="53" nodeId="6230"/>
      <Socket itemId="2"  nodeId="7960"/>
    </Sockets>
  </Spec>
</Tree>
```

Each `<Socket itemId=… nodeId=…>` points at an `<Item id=…>` block. To change a jewel, add a new `<Item id="N">` block (unique IDs above the current maximum) and repoint the socket. Only sockets of the **active spec** matter; other specs keep their own list.

An item block is plain item text, tab-indented after the tag:

```xml
<Item id="53">
			Rarity: RARE
Budget Jewel 1
Viridian Jewel
Item Level: 84
Implicits: 0
+18 to maximum Life
+10% to Chaos Resistance
</Item>
```

## 2. Skill set and config set are flat copies

```xml
<Skills activeSkillSet="4" …>
  <SkillSet id="1" title="Default">…</SkillSet>
  <SkillSet id="4" title="BUDGET">…copy of the target set…</SkillSet>
</Skills>

<Config activeConfigSet="3">
  <ConfigSet id="1" title="Default">…</ConfigSet>
  <ConfigSet id="3" title="BUDGET">…copy of the target set…</ConfigSet>
</Config>
```

Copy the sibling block verbatim, give it a new `id` and the shared `title`, then point the `active*` attribute on the parent at it. All four families must share the title or the loadout dropdown reports it incomplete.

## 3. What the shared title buys

`select_loadout({name:"BUDGET"})` then switches tree, items, skills and config together. Mixing families by hand (`select_spec` + `select_item_set`) produces plausible but wrong stats, because gems and config stay on the target's assumptions.

## 4. After the edit

`lua_reload_build()` → `select_loadout({name:"BUDGET"})` → `get_pob_warnings()`. If a warning appears, it belongs to the budget loadout, not the target — fix it there.
