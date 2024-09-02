---
title: "VIM Keyboard Shortcuts"
date: 2022-09-07T13:14:35+08:00
tags: ['Linux']
---

# VIM Keyboard Shortcuts

| **Cursor movement**             |                                                              |
| ------------------------------- | ------------------------------------------------------------ |
| h                               | move left                                                    |
| j                               | move down                                                    |
| k                               | move up                                                      |
| l                               | move right                                                   |
| w                               | jump by start of words (punctuation considered words)        |
| W                               | jump by words (spaces separate words)                        |
| e                               | jump to end of words (punctuation considered words)          |
| E                               | jump to end of words (no punctuation)                        |
| b                               | jump backward by words (punctuation considered words)        |
| B                               | jump backward by words (no punctuation)                      |
| 0                               | (zero) start of line                                         |
| ^                               | first non-blank character of line                            |
| $                               | end of line                                                  |
| G                               | Go To command (prefix with number                            |
| Note:                           | Prefix a cursor movement command with a number to repeat it. For example, 4j moves down 4 lines. |
| Insert Mode                     | Inserting/Appending text                                     |
| i                               | start insert mode at cursor                                  |
| I                               | insert at the beginning of the line                          |
| a                               | append after the cursor                                      |
| A                               | append at the end of the line                                |
| o                               | open (append) blank line below current line (no need to press return) |
| O                               | open blank line above current line                           |
| ea                              | append at end of word                                        |
| Esc                             | exit insert mode                                             |
|                                 |                                                              |
| **Editing**                     |                                                              |
| r                               | replace a single character (does not use insert mode)        |
| J                               | join line below to the current one                           |
| cc                              | change (replace) an entire line                              |
| cw                              | change (replace) to the end of word                          |
| c$                              | change (replace) to the end of line                          |
| s                               | delete character at cursor and subsitute text                |
| S                               | delete line at cursor and substitute text (same as cc)       |
| xp                              | transpose two letters (delete and paste, technically)        |
| u                               | undo                                                         |
| .                               | repeat last command                                          |
|                                 |                                                              |
| **Marking text (visual mode)**  |                                                              |
| v                               | start visual mode, mark lines, then do command (such as y-yank) |
| V                               | start Linewise visual mode                                   |
| o                               | move to other end of marked area                             |
| Ctrl+v                          | start visual block mode                                      |
| O                               | move to Other corner of block                                |
| aw                              | mark a word                                                  |
| ab                              | a () block (with braces)                                     |
| aB                              | a {} block (with brackets)                                   |
| ib                              | inner () block                                               |
| iB                              | inner {} block                                               |
| Esc                             | exit visual mode                                             |
|                                 |                                                              |
| **Visual commands**             |                                                              |
| >                               | shift right                                                  |
| <                               | shift left                                                   |
| y                               | yank (copy) marked text                                      |
| d                               | delete marked text                                           |
| ~                               | switch case                                                  |
|                                 |                                                              |
| **Cut and Paste**               |                                                              |
| yy                              | yank (copy) a line                                           |
| 2yy                             | yank 2 lines                                                 |
| yw                              | yank word                                                    |
| y$                              | yank to end of line                                          |
| p                               | put (paste) the clipboard after cursor                       |
| P                               | put (paste) before cursor                                    |
| dd                              | delete (cut) a line                                          |
| dw                              | delete (cut) the current word                                |
| x                               | delete (cut) current character                               |
|                                 |                                                              |
| **Exiting**                     |                                                              |
| :w                              | write (save) the file, but don't exit                        |
| :wq                             | write (save) and quit                                        |
| :q                              | quit (fails if anything has changed)                         |
| :q!                             | quit and throw away changes                                  |
|                                 |                                                              |
| **Search/Replace**              |                                                              |
| /pattern                        | search for pattern                                           |
| ?pattern                        | search backward for pattern                                  |
| n                               | repeat search in same direction                              |
| N                               | repeat search in opposite direction                          |
| :%s/old/new/g                   | replace all old with new throughout file                     |
| :%s/old/new/gc                  | replace all old with new throughout file with confirmations  |
|                                 |                                                              |
| **Working with multiple files** |                                                              |
| :e filename                     | Edit a file in a new buffer                                  |
| :bnext (or :bn)                 | go to next buffer                                            |
| :bprev (of :bp)                 | go to previous buffer                                        |
| :bd                             | delete a buffer (close a file)                               |
| :sp filename                    | Open a file in a new buffer and split window                 |
| ctrl+ws                         | Split windows                                                |
| ctrl+ww                         | switch between windows                                       |
| ctrl+wq                         | Quit a window                                                |
| ctrl+wv                         | Split windows vertically                                     |
|                                 |                                                              |
| **Options**                     |                                                              |
| :set all!                       |                                                              |
| :set                            |                                                              |
|                                 |                                                              |
|                                 |                                                              |
| **Resize**                      |                                                              |
| Ctrl+W +/-                      | increase/decrease height (ex. `20<C-w>+`)                    |
| Ctrl+W >/<                      | increase/decrease width (ex. `30<C-w><`)                     |
| Ctrl+W _                        | set height (ex. `50<C-w>_`)                                  |
| Ctrl+W                          | set width (ex. `50<C-w>|`)                                   |
| Ctrl+W =                        | equalize width and height of all windows                     |
|                                 |                                                              |
|                                 |                                                              |
|                                 |                                                              |
|                                 |                                                              |

## visual mode

```shell
The operators that can be used are:
        ~       switch case                                     v_~
        d       delete                                          v_d
        c       change (4)                                      v_c
        y       yank                                            v_y
        >       shift right (4)                                 v_>
        <       shift left (4)                                  v_<
        !       filter through external command (1)             v_!
        =       filter through 'equalprg' option command (1)    v_=
        gq      format lines to 'textwidth' length (1)          v_gq

The objects that can be used are:
        aw      a word (with white space)                       v_aw
        iw      inner word                                      v_iw
        aW      a WORD (with white space)                       v_aW
        iW      inner WORD                                      v_iW
        as      a sentence (with white space)                   v_as
        is      inner sentence                                  v_is
        ap      a paragraph (with white space)                  v_ap
        ip      inner paragraph                                 v_ip
        ab      a () block (with parentheses)                   v_ab
        ib      inner () block                                  v_ib
        aB      a {} block (with braces)                        v_aB
        iB      inner {} block                                  v_iB
        at      a <tag> </tag> block (withjj tags)                v_at
        it      inner <tag> </tag> block                        v_it
        a<      a <> block (with <>)                            v_a<
        i<      inner <> block                                  v_i<
        a[      a [] block (with [])                            v_a[
        i[      inner [] block                                  v_i[
        a"      a double quoted string (with quotes)            v_aquote
        i"      inner double quoted string                      v_iquote
        a'      a single quoted string (with quotes)            v_a'
        i'      inner simple quoted string                      v_i'
        a`      a string in backticks (with backticks)          v_a`
        i`      inner string in backticks                       v_i`
```

## Srolling around

```shell
ctrl+d u # one line
ctrl+e y # half of screen of text
ctrl+f b # whole screen
```

## Scrolling releative to cursor

```shell
                                                        z<CR>
z<CR>                   Redraw, line [count] at top of window (default
                        cursor line).  Put cursor at first non-blank in the
                        line.

                                                        zt
zt                      Like "z<CR>", but leave the cursor in the same
                        column.

                                                        zN<CR>
z{height}<CR>           Redraw, make window {height} lines tall.  This is
                        useful to make the number of lines small when screen
                        updating is very slow.  Cannot make the height more
                        than the physical screen height.

                                                        z.
z.                      Redraw, line [count] at center of window (default
                        cursor line).  Put cursor at first non-blank in the
                        line.

                                                        zz
zz                      Like "z.", but leavke the cursor in the same column.
                        Careful: If caps-lock is on, this command becomes
                        "ZZ": write buffer and exit!

                                                        z-
z-                      Redraw, line [count] at bottom of window (default
                        cursor line).  Put cursor at first non-blank in the
                        line.

                                                        zb
zb                      Like "z-", but leave the cursor in the same column.
```

## Find char in line

```shell
f{char}                 To [count]'th occurrence of {char} to the right.  The
                        cursor is placed on {char} inclusive.
                        {char} can be entered as a digraph digraph-arg.
                        When 'encoding' is set to Unicode, composing
                        characters may be used, see utf-8-char-arg.
                        :lmap mappings apply to {char}.  The CTRL-^ command
                        in Insert mode can be used to switch this on/off
                        i_CTRL-^.

                                                        F
F{char}                 To the [count]'th occurrence of {char} to the left.
                        The cursor is placed on {char} exclusive.
                        {char} can be entered like with the f command.

                                                        t
t{char}                 Till before [count]'th occurrence of {char} to the
                        right.  The cursor is placed on the character left of
                        {char} inclusive.
                        {char} can be entered like with the f command.

                                                        T
T{char}                 Till after [count]'th occurrence of {char} to the
                        left.  The cursor is placed on the character right of
                        {char} exclusive.
                        {char} can be entered like with the f command.

                                                        ;
;                       Repeat latest f, t, F or T [count] times. See cpo-;

                                                        ,
,                       Repeat latest f, t, F or T in opposite direction
                        [count] times. See also cpo-;
                        
```



```
:set matchpairs+=<:>
```





## VIm Tab Multiple Lines

- Enter visual Model
- Shift+dot  (ie, ".")

# Format Code

https://stackoverflow.com/questions/506075/how-do-i-fix-the-indentation-of-an-entire-file-in-vi

`=`, the indent command can take motions. So, `gg` to get the start of the file, `=` to indent, `G` to the end of the file, `gg=G`.
