# README — Shell Script Exercises

This document contains the full `shell` script **(exact, corrected)** followed by a detailed, line-by-line explanation of every *command*, *operator*, and *special symbol* used in the script. The explanations intentionally **exclude comment-only lines** from the original script and focus on showing what each executable line does and why the particular syntax/expansions/operators are used.

> **How to use this README**
> - The script block below is a ready-to-run Bash script. Save it as `shell_exercises.sh` (or any name), then `chmod +x shell_exercises.sh` and run `./shell_exercises.sh`.
> - The explanation section follows and explains each non-comment line in the script in the same order the lines appear.

---

# Script

```bash
#!/bin/bash

# ============================================================================
# SHELL SCRIPT EXERCISES - SYSTEM ADMINISTRATION & AUTOMATION
# ============================================================================

# 1. Check if file exists
check_file() {
    [ -e "$1" ] && echo "File exists" || echo "File not found"
}

# 2. Check if argument is file or directory
check_type() {
    [ -f "$1" ] && echo "Regular file"
    [ -d "$1" ] && echo "Directory"
    [ -L "$1" ] && echo "Symbolic link"
    [ ! -e "$1" ] && echo "Does not exist"
}

# 3. List empty files in current directory and subdirectories
find_empty() {
    find "${1:-.}" -type f -empty 2>/dev/null
}

# 4. Compare files and delete if same
compare_files() {
    if cmp -s "$1" "$2"; then
        echo "Files identical. Deleting $2"
        rm "$2"
    else
        echo "Files different. Differences:"
        diff "$1" "$2"
    fi
}

# 5. Multiplication table
mult_table() {
    num=${1:-5}
    for i in {1..10}; do
        echo "$num x $i = $((num * i))"
    done
}

# 6. Check and fix executable rights
fix_executables() {
    for file in *; do
        [ -f "$file" ] && [ ! -x "$file" ] && chmod +x "$file" && echo "Made $file executable"
    done
}

# 7. Arithmetic calculator
calc() {
    case "$2" in
        +) echo "$((${1} + ${3}))" ;;
        -) echo "$((${1} - ${3}))" ;;
        \*) echo "$((${1} * ${3}))" ;;
        /) echo "$((${1} / ${3}))" ;;
        %) echo "$((${1} % ${3}))" ;;
        *) echo "Invalid operator. Use: + - * / %" ;;
    esac
}

# 8. Reverse a number
reverse_num() {
    num=$1
    rev=0
    while [ $num -gt 0 ]; do
        rem=$((num % 10))
        rev=$((rev * 10 + rem))
        num=$((num / 10))
    done
    echo "$rev"
}

# 9. Convert case
convert_case() {
    str="$1"
    echo "Uppercase: ${str^^}"
    echo "Lowercase: ${str,,}"
}

# 10. Menu system
menu() {
    while true; do
        echo -e "\n=== MENU ==="
        echo "1. List files"
        echo "2. Show date"
        echo "3. Show users"
        echo "4. Show processes"
        echo "5. Exit"
        read -p "Choice: " choice
        
        case $choice in
            1) ls -l ;;
            2) date ;;
            3) who ;;
            4) ps -u $USER ;;
            5) exit 0 ;;
            *) echo "Invalid choice" ;;
        esac
    done
}

# Main Menu
main_menu() {
    while true; do
        clear
        echo "================================"
        echo "  SHELL SCRIPT EXERCISES MENU"
        echo "================================"
        echo "1.  Check if file exists"
        echo "2.  Check if file or directory"
        echo "3.  Find empty files"
        echo "4.  Compare two files"
        echo "5.  Multiplication table"
        echo "6.  Fix executable permissions"
        echo "7.  Arithmetic calculator"
        echo "8.  Reverse a number"
        echo "9.  Convert case (upper/lower)"
        echo "10. System operations menu"
        echo "0.  Exit"
        echo "================================"
        read -p "Enter choice [0-10]: " choice
        
        case $choice in
            1)
                read -p "Enter filename: " file
                check_file "$file"
                ;;
            2)
                read -p "Enter path: " path
                check_type "$path"
                ;;
            3)
                read -p "Enter directory (press Enter for current): " dir
                find_empty "$dir"
                ;;
            4)
                read -p "Enter first file: " f1
                read -p "Enter second file: " f2
                compare_files "$f1" "$f2"
                ;;
            5)
                read -p "Enter number: " num
                mult_table "$num"
                ;;
            6)
                echo "Checking and fixing executables..."
                fix_executables
                ;;
            7)
                read -p "Enter first number: " n1
                read -p "Enter operator (+,-,*,/,%): " op
                read -p "Enter second number: " n2
                calc "$n1" "$op" "$n2"
                ;;
            8)
                read -p "Enter number to reverse: " num
                reverse_num "$num"
                ;;
            9)
                read -p "Enter string: " str
                convert_case "$str"
                ;;
            10)
                menu
                ;;
            0)
                echo "Exiting..."
                exit 0
                ;;
            *)
                echo "Invalid choice!"
                ;;
        esac
        
        echo ""
        read -p "Press Enter to continue..."
    done
}

# Start the menu
main_menu
```

---

# Explanation — line by line (explanations exclude comment-only lines)

> **Note:** The left column shows the line or snippet and the right column provides a concise breakdown of the operators / tokens used and why.

### 1) `#!/bin/bash`
- `#!` is the *shebang* (hashbang). It tells the kernel what interpreter to use to run the script. Here `/bin/bash` ensures the script runs under Bash (so Bash-specific expansions like `${var^^}` will work).

### 2) `check_file() {` and `}`
- `function_name() { ... }` declares a shell function named `check_file`.
- The `{` starts the function body and `}` ends it. Newlines or semicolons separate commands inside.

### 3) `[ -e "$1" ] && echo "File exists" || echo "File not found"`
- `[` is a Bash builtin (equivalent to `test`); the closing `]` is required syntax.
- `-e "$1"` is a file test: `-e` returns true if the path in `$1` exists (file, directory, symlink, etc.).
- `"$1"` — quoting the variable: prevents word-splitting and preserves spaces/newlines in filenames.
- `&&` (AND): executes the right-hand command only if the left-hand command exited with status `0` (success). Short-circuits: if left fails, the right side won't run.
- `||` (OR): executes the right-hand command only if the left-hand command returned non-zero (failure).
- Combined effect: If `[ -e "$1" ]` is true, run `echo "File exists"`. If that whole expression (including the `&&`) fails, `|| echo "File not found"` runs. Practically this results in printing either message.

### 4) `check_type() { ... }` tests:
- `[ -f "$1" ]` — `-f` true if `$1` is a **regular file**.
- `[ -d "$1" ]` — `-d` true if `$1` is a **directory**.
- `[ -L "$1" ]` — `-L` true if `$1` is a **symbolic link**.
- `[ ! -e "$1" ]` — `!` negates the test that follows. This returns true when `$1` does **not** exist.
- Each test is independent since they are on separate lines; multiple of these tests could be true in rare cases (e.g., a symlink that also points to a regular file — behavior depends on the file system and how `test` treats the link; `-f` follows symlinks).

### 5) `find "${1:-.}" -type f -empty 2>/dev/null`
- `"${1:-.}"` is **parameter expansion with default**.
  - `${parameter:-word}` expands to `parameter` if it is set and non-null; otherwise it expands to `word`.
  - So if the user passes a directory argument, that directory is used; if they pass nothing or an empty string, it becomes `.` (current directory).
  - The quotes around the expansion protect directory names containing spaces.
- `find` options explained:
  - `-type f` → limits matches to regular files.
  - `-empty` → matches empty files (0 bytes).
- `2>/dev/null` → redirects file descriptor `2` (stderr) to `/dev/null` which discards errors (useful to hide permission-denied messages).

### 6) `if cmp -s "$1" "$2"; then` ... `fi`
- `cmp -s file1 file2` compares files and sets the exit status:
  - exit `0` if identical, `1` if different, `>1` on error.
  - `-s` suppresses normal output (silent mode) so `cmp` only sets the exit code.
- `if ...; then ... else ... fi` uses the command's exit status to select the branch.
- `rm "$2"` removes the file pointed to by `$2`. Quoting protects whitespace in filenames.
- `diff "$1" "$2"` prints textual differences between files.

### 7) `num=${1:-5}`
- Another use of parameter expansion with a default: set `num` to `$1` if provided, otherwise `5`.
- This assignment creates a local variable in the function body (it’s actually global to the script unless declared `local`).

### 8) `for i in {1..10}; do` ... `done`
- `{1..10}` is **brace expansion** performed by the shell before word-splitting. It expands to the sequence `1 2 3 ... 10`.
- `for var in list; do ... done` iterates `var` over each item in the list.

### 9) `echo "$num x $i = $((num * i))"`
- `"$num"` and `"$i"` are normal variable expansions in double quotes.
- `$(( ... ))` is **arithmetic expansion**. Inside you can use standard arithmetic operators like `+ - * / %` and it evaluates as integer arithmetic.
- Multiplication inside arithmetic expansion uses `*`. When using `*` outside arithmetic contexts it is a glob; here inside `$(( ))` it is arithmetic.

### 10) `for file in *; do` ... `done`
- `*` is **pathname expansion (globbing)**: it expands to all filenames in the current directory except hidden files (those starting with `.`).
- Each `file` receives the next filename from the glob.
- Beware: `for file in *` will loop over `*` literally when no matches exist unless `nullglob` is set; typical Bash behavior is to return the literal `*`.

### 11) `[ -f "$file" ] && [ ! -x "$file" ] && chmod +x "$file" && echo "Made $file executable"`
- `[ -f "$file" ]` checks regular file.
- `[ ! -x "$file" ]` uses `!` to negate: true if `file` is **not** executable by the current user.
- `chmod +x "$file"` changes file mode to add execute permission for relevant classes (user/group/others depending on umask and current mode).
- Chaining multiple `&&` makes the chain conditional: each command runs only if the previous one succeeded.
  - If a file is regular AND not executable, then `chmod` runs; if `chmod` succeeds, the `echo` runs.

### 12) `case "$2" in` ... `esac` in `calc()`
- `case word in pattern) commands ;; pattern2) ... ;; esac` provides pattern-based branching.
- `"$2"` is the operator argument passed in position `2` to `calc`.
- `\*)` — the `*` pattern is escaped as `\*` inside the script string so the shell doesn’t expand it as a glob during parsing; inside the actual script it appears as `\*` to ensure the case pattern is a literal `*` operator (matching the `*` operator typed by the user).
- `;;` terminates each case branch.

### 13) `echo "$((${1} + ${3}))"` and arithmetic earlier
- `$( ... )` is **command substitution** (runs a command and substitutes its stdout). In the script the inner `$(())` is arithmetic expansion; wrapping it in `$( ... )` is redundant but harmless.
- `${1}`, `${3}` — positional parameter expansions referencing the first and third argument to the function.
- Expression inside `$(( ... ))` evaluates as integer arithmetic; note that division `/` truncates toward zero for integers.

### 14) `reverse_num()` loop
- `num=$1` assigns the first positional parameter to the variable `num`.
- `rev=0` initialises the reversed number accumulator.
- `while [ $num -gt 0 ]; do` uses `-gt` (greater-than) numeric test in the `test` builtin. Important: `-gt` compares integers; if `$num` is not an integer you get an error.
- `rem=$((num % 10))` uses `%` (modulus) to get the last digit.
- `rev=$((rev * 10 + rem))` shifts the accumulator left (decimal) and adds the new digit.
- `num=$((num / 10))` integer division truncates the last digit from `num`.
- This algorithm reverses positive integers. It does not handle negative numbers, non-numeric input, or preserve leading zeros.

### 15) `convert_case()`
- `str="$1"` stores the first argument in `str` (quotes preserve whitespace).
- `${str^^}` — Bash **parameter expansion for uppercase**: transforms all letters in `str` to uppercase. This **requires Bash** (not POSIX `/bin/sh`).
- `${str,,}` — Bash expansion for lowercase.

### 16) `menu()` loop and `read -p "Choice: " choice`
- `while true; do` creates an infinite loop. `true` is a command that always exits with status `0`.
- `echo -e "\n=== MENU ==="` — `-e` enables interpretation of backslash escapes (so `\n` becomes a newline). Many shells’ `echo` implementations differ; `printf` is more portable.
- `read -p "Choice: " choice` prompts the user with the string and stores input in variable `choice`.
- `case $choice in ... esac` selects actions based on the provided choice.
- `ps -u $USER` shows processes for the `$USER` environment variable (current username). `$USER` is expanded; if unset it is empty.
- `exit 0` exits the shell with status code `0` (success).

### 17) `main_menu()` and `clear`
- `clear` is an external program or terminal control sequence that clears the terminal screen for a cleaner menu display.
- The `read -p` lines gather interactive inputs, e.g. `read -p "Enter filename: " file` stores the typed filename into the `file` variable.
- When calling functions like `check_file "$file"` we quote the argument to preserve whitespace. The function receives the string as its first positional parameter (`$1`).

### 18) Passing empty input and default parameter expansion
- The `find_empty` function uses `${1:-.}` to default to `.` if the first argument is unset **or null** (empty string). That means if the user presses Enter at the prompt (which results in an empty string), the `find_empty` call will behave as if `.` was passed.

### 19) The final `main_menu` call
- The bare `main_menu` at the end executes the function, which starts the whole interactive menu. This line begins the program's primary runtime loop.

---

# Notes, caveats & portability
- **Quoting**: the script carefully quotes variables in most places (e.g. `"$var"`). That prevents word-splitting and globbing problems when filenames contain spaces or special characters.
- **Bash-specific features**: `${str^^}` / `${str,,}` (case conversion), brace expansion `{1..10}`, and `[[`-style features are Bash-specific. The shebang `#!/bin/bash` is required. Running under `/bin/sh` (dash, ash) may fail for case conversion or other syntax.
- **Arithmetic**: `$(( ))` performs integer arithmetic only. Division truncates. For floating point math you would need `bc` or other tools.
- **Globbing**: `*` does not include hidden files (dotfiles). To include them you need a different glob like `.* *` or `shopt -s dotglob` (Bash).
- **`echo -e` portability**: Not all `echo` implementations support `-e`. Use `printf "\n..."` for portable escapes.
- **`cmp -s` and `diff`**: `cmp -s` is good to check equality by exit status. `diff` is used to show differences. `cmp -s` will still return non-zero on errors (missing files). The script does not validate arguments before calling these functions.
- **Input validation**: functions like `reverse_num` and `calc` do not validate inputs (non-integers, division by zero). Consider adding checks if you need robust behavior.

---

# Quick usage (one-liners)
- Save: `cat > shell_exercises.sh` and paste the script.
- Make executable: `chmod +x shell_exercises.sh`.
- Run: `./shell_exercises.sh`.

---

