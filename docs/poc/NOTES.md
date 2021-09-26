## `*` NOTES
This **NOTES.md** contains the standard rules for writing this structured markdown book by the [authors](./AUTHORS). Ensure formal as much as possible. ^^

```bash
           &&
         &&&&&
       &&&\/& &&&
      &&|,/  |/& &&
       &&/   /  /_&& &&
         \  {  |_____/_&&&
         {  / /  owl4ce  &&
       /}`, \{______-____/_&&&
       `{\           \_/&\_/_&&
        /}`           `&\&&\&&
  , -=-~{ .-^- _           `\&&&
         `}
         {
```

### `1` - Configuring sources
#### `1.1` - Writing standard
> 1. Ensure **one option per line**;
> 2. Sort the **backslashes** used to escaping new lines;
> 3. Use **double quotation marks** for **Cmake** **paths** and **variables**;
> 4. Use **bashism** for **similar commands**, for example `{,${HEI_TRIPLET}/}lib`;
> 5. Use `pushd` and `popd` rather than `cd` for those **require going back** to **previous directory**;
> 6. Environment variables **must** before **configure script**, except for **Cmake** and **cross-compile musl**;
> 7. The last, **all about indentation**, see the difference between **configure script** and **Cmake** in all stages.
#### `1.2` - Option priority
> 1. `prefix`
> 2. `build`
> 3. `host`
> 4. `target`
> 5. `with`
> 6. `without`
> 7. `enable`
> 8. `disable`
> 9. *misc.*
> 
> Ensure to put Cmake-specific environments before number one.

### `2` - Building sources
#### `2.1` - Writing standard
> 1. Bracket **all commands** with `time { commands; }`;
> 2. Don't make the **multiple commands inline** if they are **too long**.

### `3` - Installing sources
#### `3.1` - Writing standard
> 1. Bracket **all commands** with `time { commands; }`;
> 2. Don't make the **multiple commands inline** if they are **too long**.

### `4` - Adding and modifying sources or targets
#### `4.1` - Writing standard
> * `sed`
> 1. Use `|` rather than `/` nor `\`, except for those containing **pattern** before `s/`;
> 2. Use `-e` **per expression** for those containing **multiple expressions**;
> 3. Use `-si` instead of `-i` for those modifying **multiple files**.
> 
> * `cat > /path/to/file << EOF`
> 1. Always use **double quotation marks** "EOF" for those not containing **environment variables**, and **vice versa**.
