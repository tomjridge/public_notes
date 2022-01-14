# Sparse file API and implementation



We want to copy extents of data from one file, storing the extents adjacently in another, and maintaining some map to tell us how the new location of an extent corresponds to the old location.

So roughly there will be a destination fd, written sequentially. Then we can get the current offset from the fd, and write data to the fd, and add offset->offset bindings explicitly.

Beyond this, we can add a util to take an fd, offset, length, which copies the bytes from the fd and adds a single new binding to the offset map.

We can get the map as an in-memory value. We can write the map bindings to disk in a separate ".map" file after we have finished writing the main data.

We can open the file in read-only mode, when the map bindings can be mmap'ed (since they are now readonly) and we can use interpolation search on them. 

**Expected size of bindings:** Craig estimates there are 30M live objects, which gives 30M * 2 * 8B = 480MB, but this is needed only if there is a binding for each object. In practice we are using extents, so the expected size of the bindings is perhaps at most 200MB.

**NOTE** it may be worth making a single extent out of (extent,gap,extent), if the gap is small, since we then save on recording a binding. Probably the gap should be <=32 bytes or something like that.

In readonly mode, we probably have "read n bytes from real off into buffer"; "use map to find off"; "read extent from off" (this requires looking at the next offset in the file, or even writing the length of each extent explicitly in the main file data, so maybe we don't do that? then we depend on the encoding/decoding routines to interpret properly the amount of data we need).

**NOTE** it may be worth recording the length of each extent explicitly in the file; in the following APIs we DON'T do that

The sparse file goes through a construction phase, and is then readonly. We could use "filename.tmp" and "filename.map", and then when we close the file we could rename filename.tmp. If there is a filename.tmp, then we know .map is also temp (but we should really call it something .tmp to make clear we can delete it later). So maybe fn.tmp and fn.map.tmp, and rename fn.tmp, then fn.map.tmp; if we find fn and fn.map.tmp, we can promote fn.map.tmp to fn.map.

```ocaml
module type Small-nv-map-ii = sig (* non-volatile int->int map, fits easily in memory, stored directly in a file; small = fits easily in memory; medium = may or may not fit in memory; large = larger than can comfortably fit in memory *) end

module type OFFMAP = sig 
  type offmap
  (* a map from offset to offset, with extra functions to write to file at given position *)
  ...
  
  (** [write offmap fd off] writes the offmap to fd at given offset, and returns the number of bytes that were written *)
  val write: offmap -> fd -> int -> int
  (** [read fd off] reads the offmap from fd at the given offset *)
  val read: fd -> int -> offmap
end

(** Sparse file, write API *)
module type SPARSE_W = sig
  type t
  (** As well as <filename>, this will create <filename.map> to store the map file *)
  val create: string -> t
  val current_offset: t -> int
  val append_bytes: t -> bytes -> unit
  val append_from_fd: update_map:bool -> t -> fd -> off:int -> len:int -> unit
  
  val map_add: t -> int -> int -> unit
  
  (** Internal: Write the map to the [.map] file *)
  (* val map_flush: t -> unit *)

  (** This will flush the map and close both fds *)
  val close: t -> unit
end

(** Sparse file, read API *)
module type SPARSE_R = sig
  type t
  val open_: string -> t
  val map_find: t -> off:int -> int option
  val get_fd: t -> fd
  (** And perhaps some file IO stuff, but maybe just expose fd *)
  val read: t -> off:int -> len:int -> bytes -> unit
  val close: t -> unit
end

(** Raw API just a combination of write and read API, except that we may want to treat the map via mmap rather than loading it in memory. *)
```

