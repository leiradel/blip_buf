# Blip Buffer

This is a C implementation of the C++ library [Blip Buffer](http://www.slack.net/~ant/libs/audio.html#Blip_Buffer), kindly made available by the original author Shay Green.

The code was changed to remove the memory allocations, which are now the caller's responsibility. Shay also gave me permission to release it under the LGPL, but with an exception for static linking.

## Usage

The API is well documented and straightforward to use. There are some demo programs to check. Copy `blip_buf.h` and `blip_buf.c` to your project to build.

Also check the original [readme.txt](readme.txt).

## Version

1.1.0

## License

As a special exception, you may statically link this library into your own programs and distribute such executables under the terms of your choice, without being required to distribute the source code for your own components that are statically linked to the Library.

This exception does not, however, invalidate any other reasons why your executable file might be covered by the GNU General Public License.

This exception applies only to the code released by the author under this license. If you copy code from other sources under different licenses, the above exception does not apply to the code that you have not authored.
