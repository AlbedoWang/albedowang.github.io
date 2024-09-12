---
layout: page
title: Information Security Design and Analysis
description: Misc, Advisor - Me and Crabtux
img: # assets/img/12.jpg
importance: 1
category: TA
related_publications: False
giscus_comments: False
---

In teaching, I found that students have a very weak grasp of Python. This article focuses on explaining certain Python packages and includes writeups of some past problems.

## Interacting with servers
Let's discuss the most commonly used program for interacting with servers. Here, we use the `pwn` library to implement it. In previous courses, we mentioned `telnetlib`, but this library has been removed in Python 3.13[1], so we will also abandon it here. The `pwn` library is a powerful CTF and binary analysis tool, as its name suggests, and it can greatly assist us in solving pwn-related challenges[2]. Since this course will not cover advanced binary topics, we will only use its interaction-related functions. If you're interested, you can explore the rest of the library on your own, but we won't cover it here.

### Installation
First, we use the following command to install the `pwn` library (assuming the Python environment has already been set up):

```shell
pip install pwn
```

When "Successfully installed pwn-1.0" appears, it indicates that the installation was successful.

### Usage
Next, I will explain step by step how to implement an interaction script using the `pwn` library. For convenience, we use `from pwn import *` to import the library in our Python file.

In the program, we first set the `host` (the server IP) and `port` (the port number), then use `remote` to create a `conn` object. As an example, we initialize it as follows:

```python
host = '1.2.3.4'
port = 1234
conn = remote(host, port)
```

The `conn` object has the following commonly used functions that we frequently utilize during interactions:

1. `conn.send(data)` - Sends raw data to the server.
2. `conn.recv(n)` - Receives `n` bytes of data from the server.
3. `conn.recvline()` - Receives data until a newline character is encountered.
4. `conn.recvuntil(delimiter)` - Receives data until the specified delimiter is reached.
5. `conn.sendline(data)` - Sends data followed by a newline character.
6. `conn.interactive()` - Switches the connection to interactive mode, allowing you to manually interact with the server.

These functions are essential for managing communication in typical server interactions. Note that both `recv` and `send` handle data as bytes, so if you need to convert between `str` and `bytes`, you will need to use `encode` and `decode` accordingly, as shown below:

```python
# Sending a string, converted to bytes
conn.sendline("Hello, server".encode())

# Receiving bytes and converting them back to a string
response = conn.recvline().decode()
print(response)
``` 

This ensures smooth communication when working with strings and byte data.

## File Operations
For file reading and writing in Python, you can use the built-in `open()` function to handle files. Here’s a brief explanation of how to read from and write to files in Python:

### Writing to a File
To write data to a file, you can open it in write mode (`'w'`), append mode (`'a'`), or binary write mode (`'wb'` for binary data). Here's an example:

```python
# Writing to a file (overwrites if the file exists)
with open('output.txt', 'w') as f:
    f.write("This is some text data.\n")

# Writing bytes to a file
with open('output.bin', 'wb') as f:
    f.write(b'This is binary data.')
```

### Reading from a File
You can open a file in read mode (`'r'` for text or `'rb'` for binary data) to read its contents:

```python
# Reading from a text file
with open('input.txt', 'r') as f:
    content = f.read()
    print(content)

# Reading from a binary file
with open('input.bin', 'rb') as f:
    content = f.read()
    print(content)
```

By using `with`, you ensure that the file is automatically closed after reading or writing. This makes the file operations safe and avoids potential issues such as leaving the file open.

When dealing with file processing, there are situations where we need to read from or write to specific offsets in a file. To handle this, we use the `seek()` function to move the file pointer to the desired offset. Here's how to perform these operations:

### Writing to a Specific Offset
You can use `seek(offset)` to move the file pointer to a specific byte position before writing:

```python
# Writing to a specific offset in the file
with open('file.txt', 'r+b') as f:
    f.seek(10)  # Move to the 10th byte
    f.write(b'new data')  # Write new data at the offset
```

### Reading from a Specific Offset
Similarly, you can move the file pointer before reading:

```python
# Reading from a specific offset in the file
with open('file.txt', 'rb') as f:
    f.seek(5)  # Move to the 5th byte
    data = f.read(10)  # Read 10 bytes starting from the 5th byte
    print(data)
```

### `seek()` Modes
- `f.seek(offset, 0)` - Moves the pointer to `offset` bytes from the beginning (default).
- `f.seek(offset, 1)` - Moves the pointer to `offset` bytes from the current position.
- `f.seek(offset, 2)` - Moves the pointer to `offset` bytes from the end of the file.

These methods allow precise control over where to read or write in a file, making them useful for tasks like patching binary files or handling large data sets.

## Image Processing
The Python Imaging Library (PIL), now maintained as **Pillow**, is a powerful library for opening, manipulating, and saving images.[3] Below are the common ways to use the Pillow library for various image processing tasks.

### Installation
First, you need to install the Pillow library using the following command:

```bash
pip install Pillow
```

### Opening an Image
You can open an image using the `Image.open()` function:

```python
from PIL import Image

# Open an image file
img = Image.open('example.jpg')
img.show()  # Display the image
```

### Saving an Image
To save the image after manipulation, use the `save()` method:

```python
img.save('output.jpg')
```

### Resizing an Image
You can resize an image using the `resize()` method:

```python
# Resize the image to 200x200 pixels
resized_img = img.resize((200, 200))
resized_img.show()
```

### Cropping an Image
To crop an image, use the `crop()` method. It requires a tuple specifying the left, upper, right, and lower pixel coordinates:

```python
# Crop a rectangular section from the image
cropped_img = img.crop((100, 100, 300, 300))
cropped_img.show()
```

### Rotating an Image
You can rotate an image by a certain number of degrees using the `rotate()` method:

```python
# Rotate the image by 90 degrees
rotated_img = img.rotate(90)
rotated_img.show()
```

### Converting to Grayscale
To convert an image to grayscale, use the `convert()` method with the mode `"L"`:

```python
# Convert image to grayscale
gray_img = img.convert('L')
gray_img.show()
```

### Drawing on an Image
To draw shapes or text on an image, you can use `ImageDraw` from the Pillow library:

```python
from PIL import ImageDraw

# Open an image and create a drawing object
draw = ImageDraw.Draw(img)

# Draw a rectangle on the image
draw.rectangle((50, 50, 150, 150), outline="red", width=5)

img.show()
```

### Adding Text to an Image
You can use `ImageFont` and `ImageDraw` to add text to an image:

```python
from PIL import ImageDraw, ImageFont

# Create a drawing object
draw = ImageDraw.Draw(img)

# Load a font and add text to the image
font = ImageFont.truetype("arial.ttf", 40)
draw.text((50, 50), "Hello, World!", font=font, fill="white")

img.show()
```

Pillow provides a wide range of functionality for image manipulation, from basic tasks like resizing and cropping to more advanced operations such as filtering, blending, and even drawing shapes or text on images. PIL can also handle GIF files, including opening, processing, and saving them. Readers can explore these capabilities on their own.

## Base64 and Other Base Encodings in Python
Python's `base64` module provides functionality for encoding and decoding data in base64 and similar base encodings. Here’s a brief guide to handling base encoding and various conversions between `bytes`, `str`, `int`, and `hex`.

First, import the `base64` module:

```python
import base64
```

To encode binary data (in `bytes`) to a base64-encoded string:

```python
data = b"Hello, World!"
encoded = base64.b64encode(data)
print(encoded)  # Output: b'SGVsbG8sIFdvcmxkIQ=='
```

To decode base64-encoded data back into bytes:

```python
decoded = base64.b64decode(encoded)
print(decoded)  # Output: b'Hello, World!'
```

To work with `str`, you'll need to encode/decode between `str` and `bytes`:

```python
# Encoding str to Base64
string_data = "Hello, World!"
encoded = base64.b64encode(string_data.encode())
print(encoded)  # Output: b'SGVsbG8sIFdvcmxkIQ=='

# Decoding Base64 back to str
decoded = base64.b64decode(encoded).decode()
print(decoded)  # Output: 'Hello, World!'
```

## Bytes, String, Integer, and Hex Conversions

#### Bytes to String (`str`) Conversion

If you have a `bytes` object, you can convert it to a `str` using the `decode()` method:

```python
bytes_data = b'Hello'
string_data = bytes_data.decode()
print(string_data)  # Output: 'Hello'
```

#### String (`str`) to Bytes Conversion

To convert a `str` to `bytes`, use the `encode()` method:

```python
string_data = "Hello"
bytes_data = string_data.encode()
print(bytes_data)  # Output: b'Hello'
```

#### Integer to Bytes Conversion

You can convert an integer to a `bytes` object using the `to_bytes()` method:

```python
number = 12345
byte_data = number.to_bytes(4, byteorder='big')  # 4 bytes, big-endian
print(byte_data)  # Output: b'\x00\x00\x30\x39'
```

#### Bytes to Integer Conversion

To convert `bytes` back to an integer, use the `int.from_bytes()` method:

```python
byte_data = b'\x00\x00\x30\x39'
number = int.from_bytes(byte_data, byteorder='big')
print(number)  # Output: 12345
```

#### Hexadecimal String to Bytes

You can convert a hexadecimal string to a `bytes` object using `bytes.fromhex()`:

```python
hex_string = "48656c6c6f"
byte_data = bytes.fromhex(hex_string)
print(byte_data)  # Output: b'Hello'
```

#### Bytes to Hexadecimal String

To convert `bytes` to a hexadecimal string, use the `hex()` method:

```python
byte_data = b'Hello'
hex_string = byte_data.hex()
print(hex_string)  # Output: '48656c6c6f'
```

#### Integer to Hexadecimal String

To convert an integer to a hexadecimal string:

```python
number = 12345
hex_string = hex(number)
print(hex_string)  # Output: '0x3039'
```

#### Hexadecimal String to Integer

To convert a hexadecimal string back to an integer:

```python
hex_string = '0x3039'
number = int(hex_string, 16)
print(number)  # Output: 12345
```

### Summary of Conversions

- **Base64**: Use the `base64` module for encoding/decoding base64.
- **Bytes ↔ String**: Use `encode()` to convert `str` to `bytes` and `decode()` for the reverse.
- **Bytes ↔ Integer**: Use `to_bytes()` to convert an integer to `bytes` and `int.from_bytes()` for the reverse.
- **Bytes ↔ Hexadecimal**: Use `hex()` to convert `bytes` to hex string and `bytes.fromhex()` for the reverse.
- **Integer ↔ Hexadecimal**: Use `hex()` to convert an integer to hex string and `int()` to convert hex back to an integer.

These conversions are important for dealing with various encoding schemes and data types in Python.

The Python content required for the Misc section of this course has been fully covered in the material above. For more advanced understanding, readers are encouraged to learn on their own or reach out to the TA for assistance.

---
[1]:https://docs.python.org/3/library/telnetlib.html
[2]:https://docs.pwntools.com/en/stable/about.html#module-pwn
[3]:https://pillow.readthedocs.io/en/stable/
