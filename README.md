# Targa type 10 C - CUDA
TGA (Data type 10: Run Length Encoded, RGB images) rough implementation in C - CUDA.

TGA or TARGA format is a format for describing bitmap images.
It is capable of representing bitmaps ranging from black and white, indexed colour, and RGB colour. The format also supports various compression methods.

________________________________________________________________________________
| Offset | Length |                     Description                            |
|--------|--------|------------------------------------------------------------|
|    0   |     1  |  Number of Characters in Identification Field.             |
|        |        |                                                            |
|        |        |  This field is a one-byte unsigned integer, specifying     |
|        |        |  the length of the Image Identification Field.  Its range  |
|        |        |  is 0 to 255.  A value of 0 means that no Image            |
|        |        |  Identification Field is included.                         |
|--------|--------|------------------------------------------------------------|
|    1   |     1  |  Color Map Type.                                           |
|        |        |                                                            |
|        |        |  This field contains either 0 or 1.  0 means no color map  |
|        |        |  is included.  1 means a color map is included, but since  |
|        |        |  this is an unmapped image it is usually ignored.  TIPS    |
|        |        |  ( a Targa paint system ) will set the border color        |
|        |        |  the first map color if it is present.  Wowie zowie.       |
|--------|--------|------------------------------------------------------------|
|    2   |     1  |  Image Type Code.                                          |
|        |        |                                                            |
|        |        |  Binary 10 for this type of image.                         |
|--------|--------|------------------------------------------------------------|
|    3   |     5  |  Color Map Specification.                                  |
|        |        |                                                            |
|        |        |  Ignored if Color Map Type is 0; otherwise, interpreted    |
|        |        |  as follows:                                               |
|    3   |     2  |  Color Map Origin.                                         |
|        |        |  Integer ( lo-hi ) index of first color map entry.         |
|    5   |     2  |  Color Map Length.                                         |
|        |        |  Integer ( lo-hi ) count of color map entries.             |
|    7   |     1  |  Color Map Entry Size.                                     |
|        |        |  Number of bits in color map entry.  This value is 16 for  |
|        |        |  the Targa 16, 24 for the Targa 24, 32 for the Targa 32.   |
|--------|--------|------------------------------------------------------------|
|    8   |    10  |  Image Specification.                                      |
|        |        |                                                            |
|    8   |     2  |  X Origin of Image.                                        |
|        |        |  Integer ( lo-hi ) X coordinate of the lower left corner   |
|        |        |  of the image.                                             |
|   10   |     2  |  Y Origin of Image.                                        |
|        |        |  Integer ( lo-hi ) Y coordinate of the lower left corner   |
|        |        |  of the image.                                             |
|   12   |     2  |  Width of Image.                                           |
|        |        |  Integer ( lo-hi ) width of the image in pixels.           |
|   14   |     2  |  Height of Image.                                          |
|        |        |  Integer ( lo-hi ) height of the image in pixels.          |
|   16   |     1  |  Image Pixel Size.                                         |
|        |        |  Number of bits in a pixel.  This is 16 for Targa 16,      |
|        |        |  24 for Targa 24, and .... well, you get the idea.         |
|   17   |     1  |  Image Descriptor Byte.                                    |
|        |        |  Bits 3-0 - number of attribute bits associated with each  |
|        |        |             pixel.  For the Targa 16, this would be 0 or   |
|        |        |             1.  For the Targa 24, it should be 0.  For the |
|        |        |             Targa 32, it should be 8.                      |
|        |        |  Bit 4    - reserved.  Must be set to 0.                   |
|        |        |  Bit 5    - screen origin bit.                             |
|        |        |             0 = Origin in lower left-hand corner.          |
|        |        |             1 = Origin in upper left-hand corner.          |
|        |        |             Must be 0 for Truevision images.               |
|        |        |  Bits 7-6 - Data storage interleaving flag.                |
|        |        |             00 = non-interleaved.                          |
|        |        |             01 = two-way (even/odd) interleaving.          |
|        |        |             10 = four way interleaving.                    |
|        |        |             11 = reserved.                                 |
|--------|--------|------------------------------------------------------------|
|   18   | varies |  Image Identification Field.                               |
|        |        |  Contains a free-form identification field of the length   |
|        |        |  specified in byte 1 of the image record.  It's usually    |
|        |        |  omitted ( length in byte 1 = 0 ), but can be up to 255    |
|        |        |  characters.  If more identification information is        |
|        |        |  required, it can be stored after the image data.          |
|--------|--------|------------------------------------------------------------|
| varies | varies |  Color map data.                                           |
|        |        |                                                            |
|        |        |  If the Color Map Type is 0, this field doesn't exist.     |
|        |        |  Otherwise, just read past it to get to the image.         |
|        |        |  The Color Map Specification, describes the size of each   |
|        |        |  entry, and the number of entries you'll have to skip.     |
|        |        |  Each color map entry is 2, 3, or 4 bytes.                 |
|--------|--------|------------------------------------------------------------|
| varies | varies |  Image Data Field.                                         |
|        |        |                                                            |
|        |        |  This field specifies (width) x (height) pixels.  The      |
|        |        |  RGB color information for the pixels is stored in         |
|        |        |  packets.  There are two types of packets:  Run-length     |
|        |        |  encoded packets, and raw packets.  Both have a 1-byte     |
|        |        |  header, identifying the type of packet and specifying a   |
|        |        |  count, followed by a variable-length body.                |
|        |        |  The high-order bit of the header is "1" for the           |
|        |        |  run length packet, and "0" for the raw packet.            |
|        |        |                                                            |
|        |        |  For the run-length packet, the header consists of:        |
|        |        |      __________________________________________________    |
|        |        |      | 1 bit |   7 bit repetition count minus 1.      |    |
|        |        |      |   ID  |   Since the maximum value of this      |    |
|        |        |      |       |   field is 127, the largest possible   |    |
|        |        |      |       |   run size would be 128.               |    |
|        |        |      |-------|----------------------------------------|    |
|        |        |      |   1   |  C     C     C     C     C     C    C  |    |
|        |        |      --------------------------------------------------    |
|        |        |                                                            |
|        |        |  For the raw packet, the header consists of:               |
|        |        |      __________________________________________________    |
|        |        |      | 1 bit |   7 bit number of pixels minus 1.      |    |
|        |        |      |   ID  |   Since the maximum value of this      |    |
|        |        |      |       |   field is 127, there can never be     |    |
|        |        |      |       |   more than 128 pixels per packet.     |    |
|        |        |      |-------|----------------------------------------|    |
|        |        |      |   0   |  N     N     N     N     N     N    N  |    |
|        |        |      --------------------------------------------------    |
|        |        |                                                            |
|        |        |  For the run length packet, the header is followed by      |
|        |        |  a single color value, which is assumed to be repeated     |
|        |        |  the number of times specified in the header.  The         |
|        |        |  packet may cross scan lines ( begin on one line and end   |
|        |        |  on the next ).                                            |
|        |        |  For the raw packet, the header is followed by             |
|        |        |  the number of color values specified in the header.       |
|        |        |  The color entries themselves are two bytes, three bytes,  |
|        |        |  or four bytes ( for Targa 16, 24, and 32 ), and are       |
|        |        |  broken down as follows:                                   |
|        |        |  The 2 byte entry -                                        |
|        |        |  ARRRRRGG GGGBBBBB, where each letter represents a bit.    |
|        |        |  But, because of the lo-hi storage order, the first byte   |
|        |        |  coming from the file will actually be GGGBBBBB, and the   |
|        |        |  second will be ARRRRRGG. "A" represents an attribute bit. |
|        |        |  The 3 byte entry contains 1 byte each of blue, green,     |
|        |        |  and red.                                                  |
|        |        |  The 4 byte entry contains 1 byte each of blue, green,     |
|        |        |  red, and attribute.  For faster speed (because of the     |
|        |        |  hardware of the Targa board itself), Targa 24 image are   |
|        |        |  sometimes stored as Targa 32 images.                      |
--------------------------------------------------------------------------------
