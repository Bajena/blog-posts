Solving "UTF-8 Invalid Byte Sequence" errors in ruby

If you've landed here it means you've been hit by this message in your program. This is one of most common headaches that the data submitted by our lovely users give us.

### Short introduction to UTF-8 and other encodings
UTF-8 is, as explained in [Wikipedia](https://en.wikipedia.org/wiki/UTF-8), is a set codepoints (in simple words: numbers representing characters or formatting). Every character in UTF-8 is a *sequence* of 1 up to 4 bytes.

Apart from UTF-8 there are also other encodings like *ISO-8859-1* or *Windows-1252* - you may have seen these names before in your programming career. These encodings cover a big set of characters, including special latin characters etc.

Now, even though UTF-8 covers a huge set of characters as well it is not 100% compatible with the above mentioned encodings. Take a look at the following picture:
 - Both UTF-8 and ISO-8859-1 are ASCII compatible - the include the same codepoints for digits and latin alphabet
 - UTF-8 includes characters not present in ISO-8859-1, like the rocket emoji 🚀
 - Both UTF-8 and ISO-8859-1 include "ó" characters, but these letters are defined using different codepoints - *c585* in UTF-8 and *c5* in ISO-8859-1

![enter image description here]([https://user-images.githubusercontent.com/5732023/76835014-76d9f000-682e-11ea-8854-17874dd824d9.png](https://user-images.githubusercontent.com/5732023/76835014-76d9f000-682e-11ea-8854-17874dd824d9.png))
*Encodings compatibility*

#### Why does an UTF-8 invalid byte sequence error happen?
By default
