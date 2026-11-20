# change-graph

An append-only change log for records edited on several devices, offline or not.

Nothing is ever rewritten: every edit is a new change, and the current state is a pure fold over the log (`merge`). Each change names the record's latest changes the editing device had seen (its parents). That makes each record's history a graph, and the graph, not the devices' clocks, decides which edit came after which:

- An edit made after seeing another replaces it.
- Two different values set for the same field on devices that hadn't seen each other's edit are a **clash**. Both are kept, newest first, for a person to pick from. Nothing is silently dropped.
- Deleting is a change too (a tombstone), so a delete made while another device was still editing is caught (`deleted_while_edited`).
- The same change arriving twice counts once, so retries and syncing from several places are safe.

The crate knows nothing about storage or sync: keep the changes anywhere that can add entries and list them.

## Use

Describe a kind of record: its fields, and how an edit applies.

```rust
use change_graph::{merge, record, Edit, Record, Stamp};

#[derive(Default)]
struct Note {
    title: String,
    body: String,
}

#[derive(Clone, PartialEq)]
enum NoteEdit {
    Title(String),
    Body(String),
}

impl Record for Note {
    type Edit = NoteEdit;
    type Field = u8; // edits to the same field compete
    fn field(edit: &NoteEdit) -> u8 {
        match edit {
            NoteEdit::Title(_) => 0,
            NoteEdit::Body(_) => 1,
        }
    }
    fn apply(&mut self, edit: &NoteEdit) {
        match edit {
            NoteEdit::Title(t) => self.title = t.clone(),
            NoteEdit::Body(b) => self.body = b.clone(),
        }
    }
}

let stamp = Stamp { at: 1.0, device: "phone".into() };
let log = record(&[], "note-1", vec![Edit::Set(NoteEdit::Title("Shopping".into()))], &stamp);
let notes = merge::<Note>(&log);
assert_eq!(notes[0].record.title, "Shopping");
```

`record` (one record) and `record_all` (many records at once) make the changes for new edits, building on everything seen so far. Settling a clash is just recording the chosen value: the new change descends from both sides.

## Features

- `tsify`: derives [tsify](https://crates.io/crates/tsify) TypeScript types for `Change`, `Edit`, `Stamp` and `RecordEdit`, for apps that pass them across a wasm boundary.

## License

MIT
