# VIM

vim.rtorr.com

## ~/.vimrc
vim ~/.vimrc
set ts=2 sw=2 expandtab
set nu

## Shortcuts
w - First character of the next word
W - The same than w but for the WORD
e - Last letter of the current word, if you already are in the last letter you will be at the last letter of the next word
E - The same than e but for the WORD
b - First letter of the current word, if you already are in the first letter you will be at the first letter of the previous word
B - The same than b but for the WORD
x - Delete character under the cursor (dl does the same)
X - Delete character before the cursor (db does the same)
r - Replace the character under the cursor
D - Delete from the current position until the end of the line
d - Delete range indicated by following motion (w, e, etc). dd for the entire line
~ - Flip the case of the character under the cursor
$ - To the end of the line
0 - To the first character of the line
^ - To the first non-blank character of the line
, - Reverse the last t/T/f/F search
f - Move to the next occurence of a given character in the same line
F - Move to the previous occurence of a given character in the same line
; - Repeat the last t/T/f/F search
T - Move to one character before the previous occurence of a given character in the same line
t - Move to one character before the next occurence of a given character in the same line
z - Scroll screen so the cursor current position will be at the top (t), bottom (b) or middle (z)
% - Move to the matching bracket or parentheses
G - Move to the last line of the text (soft beginning of line) or to the given [count] line number
g - Use 'gg' to move to the first line of the text (soft beginning of the line)
/ - Search for text on the file