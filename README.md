# Scalpel deleted module reproducer

Deleting a module marks its parent as a `DIRECT` `SOURCE_CHANGE`, which pulls the parent's other children and their dependents into the affected set.

## Project layout

```
root
├── libs          (aggregator + parent of lib-a, lib-b, lib-gone)
│   ├── lib-a
│   ├── lib-b
│   └── lib-gone  <- deleted on the remove-lib-gone branch; nothing depends on it
├── app           <- depends on lib-a
└── standalone
```

The `remove-lib-gone` branch deletes `libs/lib-gone` and removes it from the `<modules>` of `libs/pom.xml`.

## Reproduce

```bash
git clone https://github.com/jamesnetherton/scalpel-deletion-reproducer.git
cd scalpel-deletion-reproducer
git checkout remove-lib-gone
mvn validate -Dscalpel.enabled=true -Dscalpel.mode=report -Dscalpel.baseBranch=origin/main
jq '{unmatchedPomPaths, affectedModules: [.affectedModules[] | {artifactId, category, reasons}]}' target/scalpel-report.json
```

### Actual

```
unmatchedPomPaths: ["libs/lib-gone/pom.xml"]

libs   DIRECT      SOURCE_CHANGE
lib-a  DOWNSTREAM  DOWNSTREAM_DEPENDENT
lib-b  DOWNSTREAM  DOWNSTREAM_DEPENDENT
app    DOWNSTREAM  DOWNSTREAM_DEPENDENT
```

### Expected

`lib-a`, `lib-b` and `app` are not affected. Nothing in the reactor at HEAD depends on `lib-gone`.

## Control

Excluding the deleted module's files results in no affected modules, so the trigger is the deleted files being attributed to `libs`, rather than the `<modules>` change in `libs/pom.xml`:

```bash
mvn validate -Dscalpel.enabled=true -Dscalpel.mode=report -Dscalpel.baseBranch=origin/main '-Dscalpel.excludePaths=libs/lib-gone/**'
```
