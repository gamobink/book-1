## Example: pretty-printing a table

One common programming task is displaying tabular data.  In this
example, will go over the design of a simple library to do just that.

We'll start with the interface.  The code will go in a new module
called `Text_table` whose mli contains just the following function:

~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
(* [render headers rows] returns a string containing a formatted
   text table, using unix-style newlines as separators *)
val render
   :  string list         (* header *)
   -> string list list    (* data *)
   -> string
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

If you invoke `render` as follows:

~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
let () =
  print_string (Text_table.render
     ["language";"architect";"first release"]
     [ ["Lisp" ;"John McCarthy" ;"1958"] ;
       ["C"    ;"Dennis Ritchie";"1969"] ;
       ["ML"   ;"Robin Milner"  ;"1973"] ;
       ["OCaml";"Xavier Leroy"  ;"1996"] ;
     ])
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

you'll get the following output:

~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
| language | architect      | first release |
|----------+----------------+---------------|
| Lisp     | John McCarthy  | 1958          |
| C        | Dennis Ritchie | 1969          |
| ML       | Robin Milner   | 1973          |
| OCaml    | Xavier Leroy   | 1996          |
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Now that we know what `render` is supposed to do, let's dive into the
implementation.

### Computing the widths

To render the rows of the table, we'll first need the width of the
widest entry in each column.  The following function does just that.

~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
let max_widths header rows =
  let to_lengths l = List.map ~f:String.length l in
  List.fold rows
    ~init:(to_lengths header)
    ~f:(fun acc row ->
      List.map2_exn ~f:Int.max acc (to_lengths row))
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

In the above we define a helper function, `to_lengths` which uses
`List.map` and `String.length` to convert a list of strings to a list
of string lengths.  Then, starting with the lengths of the headers, we
use `List.fold` to join in the lengths of the elements of each row by
`max`'ing them together elementwise.


Note that this code will throw an exception if any of the rows has a
different number of entries than the header.  In particular,
`List.map2_exn` throws an exception when its arguments have mismatched
lengths.

### Rendering the rows

Now we need to write the code to render a single row.  There are
really two different kinds of rows that need to be rendered: an
ordinary row:

~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
| Lisp     | John McCarthy  | 1962          |
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

and a separator row:

~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
|----------+----------------+---------------|
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Let's start with the separator row, which we can generate as follows:

~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
let render_separator widths =
  let pieces = List.map widths
    ~f:(fun w -> String.make (w + 2) '-')
  in
  "|" ^ String.concat ~sep:"+" pieces ^ "|"
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

We need the extra two-characters for each entry to account for the one
character of padding on each side of a string in the table.

#### Note: Performance of `String.concat` and `^`

> In the above, we're using two different ways of concatenating
> strings, `String.concat`, which operates on lists of strings, and
> `^`, which is a pairwise operator.  You should avoid `^` for joining
> long numbers of strings, since, it allocates a new string every time
> it runs.  Thus, the following code:
>
> ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
> let s = "." ^ "."  ^ "."  ^ "."  ^ "."  ^ "."  ^ "."
> ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
>
> will allocate a string of length 2, 3, 4, 5, 6 and 7, whereas this
> code:
>
> ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
> let s = String.concat [".";".";".";".";".";".";"."]
> ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
>
> allocates one string of size 7, as well as a list of length 7.  At
> these small sizes, the differences don't amount ot much, but for
> assembling of large strings, it can be a serious performance issue.

We can write a very similar piece of code for rendering the data in a
an ordinary row.

~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
let pad s length =
  if String.length s >= length then s
  else s ^ String.make (length - String.length s) ' '

let render_row row widths =
  let pieces = List.map2 row widths
    ~f:(fun s width -> " " ^ pad s width ^ " ")
  in
  "|" ^ String.concat ~sep:"|" pieces ^ "|"
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

You might note that `render_row` and `render_separator` share a bit of
structure.  We can improve the code a bit by factoring that repeated
structure out:

~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
let decorate_row ~sep row = "|" ^ String.concat ~sep row ^ "|"

let render_row widths row =
  decorate_row ~sep:"|"
    (List.map2_exn row widths ~f:(fun s w -> " " ^ pad s w ^ " "))

let render_separator widths =
  decorate_row ~sep:"+"
    (List.map widths ~f:(fun width -> String.make (width + 2) '-'))
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

### Bringing it all together

And now we can write the function for rendering a full table.

~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
let render header rows =
  let widths = max_widths header rows in
  String.concat ~sep:"\n"
    (render_row widths header
     :: render_separator widths
     :: List.map rows ~f:(fun row -> render_row widths row)
    )
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Now, let's think about how you might actually use this interface in
practice.  Usually, when you have data to render in a table, that data
comes in the form of a list of objects of some sort, where you need to
extract data from each record for each of the columns.  So, imagine
that you start off with a record type for representing information
about a given programming language:

~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
type style =
    Object_oriented | Functional | Imperative | Logic
type prog_lang = { name: string;
                   architect: string;
                   year_released: int;
                   style: style list;
                 }
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

If we then wanted to render a table from a list of languages, we might
write something like this:

~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
let print_langs langs =
   let headers = ["name";"architect";"year released"] in
   let to_row lang =
     [lang.name; lang.architect; Int.to_string lang.year_released ]
   in
   print_string (Text_table.render headers (List.map ~f:to_row langs))
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

This works fine, but as you consider more complicated tables with more
columns, it becomes easier to make the mistake of having a mismatch
between `headers` and `to_row`.  Also, reordering columns or rendering
with a subset of the columns becomes awkward, because changes need to
be made in two places.

We can improve the table API by adding a type which is a first-class
representative for a column.  We'd add the following to the interface
of `Text_table`:

~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
(** An ['a column] is a specification of a column for rending a table
    of values of type ['a] *)
type 'a column

(** [column header to_entry] returns a new column given a header and a
    function for extracting the text entry from the data associated
    with a row *)
val column : string -> ('a -> string) -> 'a column

(** [column_render columns rows] Renders a table with the specified
    columns and rows *)
val column_render :
  'a column list -> 'a list -> string
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Thus, the `column` functions creates a `column` from a header string
and a function for extracting the text for that column associated with
a given row.  Implementing this interface is quite simple:

~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
type 'a column = string * ('a -> string)
let column header to_string = (header,to_string)

let column_render columns rows =
  let header = List.map columns ~f:fst in
  let rows = List.map rows ~f:(fun row ->
    List.map columns ~f:(fun (_,to_string) -> to_string row))
  in
  render header rows
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

And we can rewrite `print_langs` to use this new interface as follows.

~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
let print_langs langs =
  let colums =
    [ Text_table.column "Name"      (fun x -> x.name);
      Text_table.column "Architect" (fun x -> x.architect);
      Text_table.column "Year Released"
         (fun x -> Int.to_string x.year_released);
    ]
  in
  print_string (Text_table.column_render columns langs)
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

The code is a bit longer, but it's also less error prone.  In
particular, several errors that might be made by the user are now
ruled out by the type system.  For example, it's no longer possible
for the length of the header and the lengths of the rows to be
mismatched.

The simple column-based interface described here is a good starting
for building a richer API.  You could for example build specialized
colums with different formatting and alignment rules, which is easier
to do with this interface than with the original one based on passing
in lists-of-lists.