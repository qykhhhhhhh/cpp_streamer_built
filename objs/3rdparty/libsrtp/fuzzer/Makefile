# Makefile for libSRTP fuzzer

srcdir = .
top_srcdir = ..
top_builddir = ..


CC	= gcc
CXX	= g++
INCDIR	= -Iinclude -I$(srcdir)/include -I$(top_srcdir)/include -I$(top_srcdir)/crypto/include/
DEFS	= -DHAVE_CONFIG_H
CPPFLAGS=  -g
CXXFLAGS= -g -O2 -g
CFLAGS	= -fPIC -Wall -pedantic -Wstrict-prototypes -O3 -fexpensive-optimizations -funroll-loops -I/home/root/cpp_streamer/objs/output/include -g
LIBS	= -L/home/root/cpp_streamer/objs/output/lib -lcrypto -ldl -pthread 
COMPILE = $(CC) $(DEFS) $(INCDIR) $(CPPFLAGS) $(CFLAGS)
COMPILECXX = $(CXX) $(DEFS) $(INCDIR) $(CPPFLAGS) $(CXXFLAGS)
CRYPTOLIB = -lsrtp2

.PHONY: clean

all : srtp-fuzzer

mt19937.o: mt19937.cpp
	$(COMPILECXX) -c -std=c++11 mt19937.cpp -o mt19937.o
fuzzer.o: fuzzer.c fuzzer.h testmem.h
	$(COMPILE) fuzzer.c -c -o fuzzer.o
testmem.o: testmem.c
	$(COMPILE) -O0 testmem.c -c -o testmem.o
srtp-fuzzer: fuzzer.o mt19937.o testmem.o
	$(COMPILECXX) -L. -L.. fuzzer.o mt19937.o testmem.o $(LIBFUZZER) $(CRYPTOLIB) $(LIBS) -o srtp-fuzzer

clean:
	rm -rf srtp-fuzzer *.o
