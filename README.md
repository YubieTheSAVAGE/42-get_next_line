# 📖 get_next_line

A 42 School project that implements a function to read a line from a file descriptor, one line at a time.

---

## 🎯 The Concept

The goal of `get_next_line` is simple yet powerful: create a function that reads and returns **one line at a time** from a file descriptor. Each subsequent call returns the next line until EOF (End of File) is reached.

### Why is this challenging?

1. **Unknown line lengths** — Lines can be any size, from empty to thousands of characters
2. **Efficient buffering** — Reading byte-by-byte is slow; reading too much wastes memory
3. **State persistence** — The function must remember where it left off between calls
4. **Multiple file descriptors** (bonus) — Handle several files simultaneously without mixing their states

---

## 🧠 Core Concepts

### Static Variables

The magic behind `get_next_line` lies in **static variables**. Unlike regular local variables that are destroyed when a function returns, static variables:

- Persist their value between function calls
- Are initialized only once
- Maintain state throughout the program's lifetime

```c
char *get_next_line(int fd)
{
    static char *buffer;  // Retains value between calls!
    // ...
}
```

### The Buffer Strategy

The function uses a configurable `BUFFER_SIZE` to read chunks of data:

```
┌─────────────────────────────────────────────────────────┐
│  File Content: "Hello\nWorld\nThis is GNL\n"            │
└─────────────────────────────────────────────────────────┘
                           │
                           ▼
┌──────────────────────────────────────────────────────────┐
│  BUFFER_SIZE = 5                                         │
│  ─────────────────────────────────────────────────────── │
│  Read 1: "Hello"  → Found '\n' → Return "Hello\n"        │
│  Read 2: "\nWorl" → Found '\n' → Return "World\n"        │
│  Read 3: "d\nThi" → Continue...                          │
│  ...                                                     │
└──────────────────────────────────────────────────────────┘
```

---

## 🔧 How It Works

### The Algorithm Flow

```
┌─────────────────────────────────────────────────────────────┐
│                     get_next_line(fd)                        │
└─────────────────────────────────────────────────────────────┘
                             │
                             ▼
                   ┌─────────────────┐
                   │  Validate fd &  │
                   │  BUFFER_SIZE    │
                   └────────┬────────┘
                            │
                            ▼
              ┌─────────────────────────────┐
              │   get_read(fd, buffer)      │
              │   ─────────────────────     │
              │   Read until '\n' found     │
              │   or EOF reached            │
              └─────────────┬───────────────┘
                            │
                            ▼
              ┌─────────────────────────────┐
              │      get_line(buffer)       │
              │   ─────────────────────     │
              │   Extract line up to '\n'   │
              └─────────────┬───────────────┘
                            │
                            ▼
              ┌─────────────────────────────┐
              │     get_rest(buffer)        │
              │   ─────────────────────     │
              │   Save remaining content    │
              │   for next call             │
              └─────────────┬───────────────┘
                            │
                            ▼
                   ┌─────────────────┐
                   │   Return line   │
                   └─────────────────┘
```

### Key Functions Explained

#### `get_next_line(int fd)` — The Main Function

```c
char *get_next_line(int fd)
{
    static char *buffer;      // Persists between calls
    char        *line;

    // 1. Validate input
    if (fd < 0 || BUFFER_SIZE <= 0)
        return (NULL);
    
    // 2. Read from file until we have a complete line
    buffer = get_read(fd, buffer);
    
    // 3. Extract the line (up to and including '\n')
    line = get_line(buffer);
    
    // 4. Keep the rest for the next call
    buffer = get_rest(buffer, ft_strlen(buffer));
    
    return (line);
}
```

#### `get_read(int fd, char *buffer)` — The Reader

Reads from the file descriptor in chunks of `BUFFER_SIZE` until:
- A newline character `'\n'` is found, OR
- End of file (EOF) is reached

```c
char *get_read(int fd, char *buffer)
{
    char *read_buffer;
    int  read_bytes;

    read_buffer = malloc((BUFFER_SIZE + 1) * sizeof(char));
    while (1)
    {
        read_bytes = read(fd, read_buffer, BUFFER_SIZE);
        if (read_bytes <= 0)
            break;
        read_buffer[read_bytes] = '\0';
        buffer = ft_strjoin(buffer, read_buffer);  // Append to existing buffer
        if (ft_strnl(buffer))                       // Check for newline
            break;
    }
    free(read_buffer);
    return (buffer);
}
```

#### `get_line(char *buffer)` — Line Extractor

Extracts everything from the start of the buffer up to and including `'\n'`:

```c
char *get_line(char *buffer)
{
    size_t i = 0;
    
    while (buffer[i] && buffer[i] != '\n')
        i++;
    
    return ft_substr(buffer, 0, i + 1);  // Include the '\n'
}
```

#### `get_rest(char *buffer)` — Remainder Keeper

Saves everything after the newline for the next call:

```c
char *get_rest(char *buffer, size_t line_size)
{
    size_t i = 0;
    char   *rest;
    
    while (buffer[i] && buffer[i] != '\n')
        i++;
    
    rest = ft_substr(buffer, i + 1, line_size);  // Everything after '\n'
    free(buffer);
    return (rest);
}
```

---

## ⭐ Bonus: Multiple File Descriptors

The bonus version handles multiple files simultaneously by using an **array of static buffers**:

```c
char *get_next_line(int fd)
{
    static char *buffer[MAX_FD];  // One buffer per file descriptor!
    
    // Use buffer[fd] instead of buffer
    buffer[fd] = get_read(fd, buffer[fd]);
    // ...
}
```

This allows interleaved reading from different files:

```c
int fd1 = open("file1.txt", O_RDONLY);
int fd2 = open("file2.txt", O_RDONLY);

get_next_line(fd1);  // Line 1 from file1
get_next_line(fd2);  // Line 1 from file2
get_next_line(fd1);  // Line 2 from file1
get_next_line(fd2);  // Line 2 from file2
```

---

## 📁 Project Structure

```
42-get_next_line/
├── get_next_line.c          # Main function (mandatory)
├── get_next_line.h          # Header file (mandatory)
├── get_next_line_utils.c    # Helper functions (mandatory)
├── get_next_line_bonus.c    # Multi-fd version (bonus)
├── get_next_line_bonus.h    # Header file (bonus)
└── get_next_line_utils_bonus.c  # Helper functions (bonus)
```

---

## 🛠️ Helper Functions

| Function | Description |
|----------|-------------|
| `ft_strlen` | Returns the length of a string |
| `ft_strdup` | Duplicates a string |
| `ft_strjoin` | Concatenates two strings (frees s1) |
| `ft_substr` | Extracts a substring |
| `ft_strnl` | Checks if string contains `'\n'` |

---

## 🚀 Usage

### Compilation

```bash
# Mandatory part
cc -Wall -Wextra -Werror -D BUFFER_SIZE=42 get_next_line.c get_next_line_utils.c main.c

# Bonus part
cc -Wall -Wextra -Werror -D BUFFER_SIZE=42 get_next_line_bonus.c get_next_line_utils_bonus.c main.c
```

### Example Usage

```c
#include "get_next_line.h"
#include <fcntl.h>
#include <stdio.h>

int main(void)
{
    int   fd;
    char  *line;

    fd = open("example.txt", O_RDONLY);
    if (fd == -1)
        return (1);
    
    while ((line = get_next_line(fd)) != NULL)
    {
        printf("%s", line);
        free(line);  // Don't forget to free!
    }
    
    close(fd);
    return (0);
}
```

### Testing with Different Buffer Sizes

```bash
# Small buffer (more read calls)
cc -D BUFFER_SIZE=1 get_next_line.c get_next_line_utils.c main.c

# Large buffer (fewer read calls)
cc -D BUFFER_SIZE=10000 get_next_line.c get_next_line_utils.c main.c
```

---

## ⚠️ Edge Cases Handled

- ✅ Empty file
- ✅ File with no newline at EOF
- ✅ Very long lines
- ✅ Invalid file descriptor
- ✅ `BUFFER_SIZE` of 1
- ✅ Very large `BUFFER_SIZE`
- ✅ Reading from stdin (fd = 0)
- ✅ Multiple file descriptors (bonus)

---

## 💡 Key Takeaways

1. **Static variables** are essential for maintaining state between function calls
2. **Buffered I/O** is crucial for performance — don't read byte by byte
3. **Memory management** is critical — always free what you allocate
4. **Edge cases matter** — handle EOF, empty files, and invalid inputs gracefully

---

## 📝 Return Values

| Return | Meaning |
|--------|---------|
| `char *` | The next line read from the file descriptor |
| `NULL` | EOF reached or error occurred |

---

## 👤 Author

**aboudiba** — 42 Student

---

<p align="center">
  <i>Made with ❤️ at 1337</i>
</p>

