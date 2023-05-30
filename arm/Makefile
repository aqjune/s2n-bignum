#############################################################################
# Copyright Amazon.com, Inc. or its affiliates. All Rights Reserved.
# SPDX-License-Identifier: Apache-2.0 OR ISC
#############################################################################

OSTYPE_RESULT=$(shell uname -s)
ARCHTYPE_RESULT=$(shell uname -m)

# Assembler directives that mark symbols as .hidden
# or .private_extern can be enabled by passing
# in the S2N_BN_HIDE_SYMBOLS parameter as:
#
#    make S2N_BN_HIDE_SYMBOLS=1
#

ifeq ($(S2N_BN_HIDE_SYMBOLS),1)
SYMBOL_HIDING=-DS2N_BN_HIDE_SYMBOLS=1
else
SYMBOL_HIDING=
endif


# Add explicit language input parameter to cpp, otherwise the use of #n for
# numeric literals in ARM code is a problem when used inside #define macros
# since normally that means stringization.
#
# Some clang-based preprocessors seem to behave differently, and get confused
# by single-quote characters in comments, so we eliminate // comments first.

ifeq ($(OSTYPE_RESULT),Darwin)
PREPROCESS=sed -e 's/\/\/.*//' | $(CC) -E -I../include $(SYMBOL_HIDING) -xassembler-with-cpp -
else
PREPROCESS=$(CC) -E -I../include $(SYMBOL_HIDING) -xassembler-with-cpp -
endif

# Generally GNU-type assemblers are happy with multiple instructions on
# a line, but we split them up anyway just in case.

SPLIT=tr ';' '\n'

# If actually on an ARM8 machine, just use the assembler (as). Otherwise
# use a cross-assembling version so that the code can still be assembled
# and the proofs checked against the object files (though you won't be able
# to run code without additional emulation infrastructure). For the clang
# version on OS X we just add the "-arch arm64" option. For the Linux/gcc
# toolchain we assume the presence of the special cross-assembler. This
# can be installed via something like:
#
#  sudo apt-get install binutils-aarch64-linux-gnu

ifeq ($(ARCHTYPE_RESULT),aarch64)
ASSEMBLE=as
else
ifeq ($(ARCHTYPE_RESULT),arm64)
ASSEMBLE=as
else
ifeq ($(OSTYPE_RESULT),Darwin)
ASSEMBLE=as -arch arm64
else
ASSEMBLE=aarch64-linux-gnu-as
endif
endif
endif

# List of object files for point operations and bignum operations

POINT_OBJ = curve25519/curve25519_ladderstep.o \
            curve25519/curve25519_ladderstep_alt.o \
            curve25519/curve25519_pxscalarmul.o \
            curve25519/curve25519_pxscalarmul_alt.o \
            curve25519/curve25519_x25519.o \
            curve25519/curve25519_x25519_alt.o \
            curve25519/curve25519_x25519_byte.o \
            curve25519/curve25519_x25519_byte_alt.o \
            curve25519/curve25519_x25519base.o \
            curve25519/curve25519_x25519base_alt.o \
            curve25519/curve25519_x25519base_byte.o \
            curve25519/curve25519_x25519base_byte_alt.o \
            curve25519/edwards25519_epadd.o \
            curve25519/edwards25519_epadd_alt.o \
            curve25519/edwards25519_epdouble.o \
            curve25519/edwards25519_epdouble_alt.o \
            curve25519/edwards25519_pdouble.o \
            curve25519/edwards25519_pdouble_alt.o \
            curve25519/edwards25519_pepadd.o \
            curve25519/edwards25519_pepadd_alt.o \
            curve25519/edwards25519_scalarmulbase.o \
            curve25519/edwards25519_scalarmulbase_alt.o \
            p256/p256_montjadd.o \
            p256/p256_montjdouble.o \
            p256/p256_montjmixadd.o \
            p384/p384_montjadd.o \
            p384/p384_montjdouble.o \
            p384/p384_montjmixadd.o \
            p521/p521_jadd.o \
            p521/p521_jdouble.o \
            p521/p521_jmixadd.o \
            secp256k1/secp256k1_jadd.o \
            secp256k1/secp256k1_jdouble.o \
            secp256k1/secp256k1_jmixadd.o \
            sm2/sm2_montjadd.o \
            sm2/sm2_montjmixadd.o \
            sm2/sm2_montjdouble.o

BIGNUM_OBJ = fastmul/bignum_emontredc_8n_neon.o \
             fastmul/bignum_kmul_16_32_neon.o \
             fastmul/bignum_kmul_32_64_neon.o \
             fastmul/bignum_ksqr_16_32_neon.o \
             fastmul/bignum_ksqr_32_64_neon.o \
             fastmul/bignum_mul_8_16_neon.o \
             fastmul/bignum_sqr_8_16_neon.o

OBJ = $(BIGNUM_OBJ)

%.o : %.S ; cat $< | $(PREPROCESS) | $(SPLIT) | $(ASSEMBLE) -o $@ -

libs2nbignum.a: $(OBJ) ; ar -rc libs2nbignum.a $(OBJ)

clean:; rm -f libs2nbignum.a */*.o */*.correct

# Proof-related parts
#
# The proof files are all independent, though each one loads the
# same common infrastructure "base.ml". So you can potentially
# run the proofs in parallel for more speed, e.g.
#
#    nohup make -j 16 proofs &
#
# On debian, the underlying prover HOL Light can be installed as a package
# by something like
#
#    sudo apt-get install hol-light
#
# for which you would set this below:
#
#    HOLDIR=/usr/share/hol-light
#
# If you build hol-light yourself (see https://github.com/jrh13/hol-light)
# in your home directory, and do "make" inside the subdirectory hol-light,
# then the following HOLDIR setting should be right:

#HOLDIR?=$(HOME)/hol-light
HOLLIGHT:=/home/ubuntu/s2n-bignum-github/hollight #HOLLIGHT_DIR=$(HOLDIR) ocaml -init $(HOLDIR)/hol.ml

PROOFS = $(OBJ:.o=.correct)

# Cases where a proof uses other proofs for lemmas and/or subroutines

curve25519/curve25519_x25519.correct: generic/bignum_modinv.o proofs/curve25519_x25519.ml curve25519/curve25519_x25519.o ; (cd ..; (echo 'loadt "arm/proofs/base.ml";;'; echo 'loadt "arm/proofs/curve25519_x25519.ml";;') | $(HOLLIGHT) 2>&1) >$@
curve25519/curve25519_x25519_alt.correct: generic/bignum_modinv.o proofs/curve25519_x25519_alt.ml curve25519/curve25519_x25519_alt.o ; (cd ..; (echo 'loadt "arm/proofs/base.ml";;'; echo 'loadt "arm/proofs/curve25519_x25519_alt.ml";;') | $(HOLLIGHT) 2>&1) >$@
curve25519/curve25519_x25519_byte.correct: generic/bignum_modinv.o proofs/curve25519_x25519_byte.ml curve25519/curve25519_x25519_byte.o ; (cd ..; (echo 'loadt "arm/proofs/base.ml";;'; echo 'loadt "arm/proofs/curve25519_x25519_byte.ml";;') | $(HOLLIGHT) 2>&1) >$@
curve25519/curve25519_x25519_byte_alt.correct: generic/bignum_modinv.o proofs/curve25519_x25519_byte_alt.ml curve25519/curve25519_x25519_byte_alt.o ; (cd ..; (echo 'loadt "arm/proofs/base.ml";;'; echo 'loadt "arm/proofs/curve25519_x25519_byte_alt.ml";;') | $(HOLLIGHT) 2>&1) >$@
curve25519/curve25519_x25519base.correct: generic/bignum_modinv.o proofs/curve25519_x25519base.ml curve25519/curve25519_x25519base.o ; (cd ..; (echo 'loadt "arm/proofs/base.ml";;'; echo 'loadt "arm/proofs/curve25519_x25519base.ml";;') | $(HOLLIGHT) 2>&1) >$@
curve25519/curve25519_x25519base_alt.correct: generic/bignum_modinv.o proofs/curve25519_x25519base_alt.ml curve25519/curve25519_x25519base_alt.o ; (cd ..; (echo 'loadt "arm/proofs/base.ml";;'; echo 'loadt "arm/proofs/curve25519_x25519base_alt.ml";;') | $(HOLLIGHT) 2>&1) >$@
curve25519/curve25519_x25519base_byte.correct: generic/bignum_modinv.o proofs/curve25519_x25519base_byte.ml curve25519/curve25519_x25519base_byte.o ; (cd ..; (echo 'loadt "arm/proofs/base.ml";;'; echo 'loadt "arm/proofs/curve25519_x25519base_byte.ml";;') | $(HOLLIGHT) 2>&1) >$@
curve25519/curve25519_x25519base_byte_alt.correct: generic/bignum_modinv.o proofs/curve25519_x25519base_byte_alt.ml curve25519/curve25519_x25519base_byte_alt.o ; (cd ..; (echo 'loadt "arm/proofs/base.ml";;'; echo 'loadt "arm/proofs/curve25519_x25519base_byte_alt.ml";;') | $(HOLLIGHT) 2>&1) >$@
generic/bignum_modexp.correct: generic/bignum_amontifier.correct generic/bignum_amontmul.correct generic/bignum_demont.correct generic/bignum_mux.correct proofs/bignum_modexp.ml generic/bignum_modexp.o ; (cd ..; (echo 'loadt "arm/proofs/base.ml";;'; echo 'loadt "arm/proofs/bignum_modexp.ml";;') | $(HOLLIGHT) 2>&1) >$@

# All other other instances are standalone

curve25519/%.correct: proofs/%.ml curve25519/%.o ; (cd ..; (echo 'loadt "arm/proofs/base.ml";;'; echo 'loadt "arm/proofs/$*.ml";;') | $(HOLLIGHT) 2>&1) >$@
fastmul/%.correct: proofs/%.ml fastmul/%.o ; (cd ..; (echo 'loadt "arm/proofs/base.ml";;'; echo 'loadt "arm/proofs/$*.ml";;') | $(HOLLIGHT) 2>&1) >$@
generic/%.correct: proofs/%.ml generic/%.o ; (cd ..; (echo 'loadt "arm/proofs/base.ml";;'; echo 'loadt "arm/proofs/$*.ml";;') | $(HOLLIGHT) 2>&1) >$@
p256/%.correct: proofs/%.ml p256/%.o ; (cd ..; (echo 'loadt "arm/proofs/base.ml";;'; echo 'loadt "arm/proofs/$*.ml";;') | $(HOLLIGHT) 2>&1) >$@
p384/%.correct: proofs/%.ml p384/%.o ; (cd ..; (echo 'loadt "arm/proofs/base.ml";;'; echo 'loadt "arm/proofs/$*.ml";;') | $(HOLLIGHT) 2>&1) >$@
p521/%.correct: proofs/%.ml p521/%.o ; (cd ..; (echo 'loadt "arm/proofs/base.ml";;'; echo 'loadt "arm/proofs/$*.ml";;') | $(HOLLIGHT) 2>&1) >$@
secp256k1/%.correct: proofs/%.ml secp256k1/%.o ; (cd ..; (echo 'loadt "arm/proofs/base.ml";;'; echo 'loadt "arm/proofs/$*.ml";;') | $(HOLLIGHT) 2>&1) >$@
sm2/%.correct: proofs/%.ml sm2/%.o ; (cd ..; (echo 'loadt "arm/proofs/base.ml";;'; echo 'loadt "arm/proofs/$*.ml";;') | $(HOLLIGHT) 2>&1) >$@

run_proofs: $(PROOFS);

proofs: run_proofs ; grep -r -i "error" --include "*.correct" ; [ `grep 'let [A-Z_0-9]*_SUBROUTINE_CORRECT' proofs/*.ml | wc -l` -eq  `grep 'SUBROUTINE_CORRECT : thm' */*.correct  | cut -f2 -d' ' | sort | uniq | wc -l` ]
