# Signature tests

Date: 2026-10-01. Method: grep -rE --include='*.rs' -c <pattern> over five real codebases
(raydium-clmm 79 rs files, raydium-cp-swap 43, manifest 108, hylo 166, metadao futarchy 192).
Counts are total matching lines. A hit scopes the search; it is not evidence.

| Vector | Signature | clmm | cp-swap | manifest | hylo | metadao |
|--------|-----------|------|---------|----------|------|---------|
| VS1 | Signer<'info> | 50 | 16 | 0 | 31 | 113 |
| VS1 | /// CHECK or UncheckedAccount | 73 | 41 | 0 | 12 | 436 |
| VS1 | is_signer | 0 | 2 | 17 | 29 | 9 |
| VS2 | try_from_slice family | 19 | 11 | 22 | 5 | 1 |
| VS2 | .owner | 54 | 34 | 23 | 8 | 9 |
| VS3 | has_one/constraint/address | 112 | 51 | 11 | 24 | 216 |
| VS4 | seeds/bump | 73 | 35 | 1 | 104 | 197 |
| VS4 | find/create_program_address | 56 | 13 | 17 | 16 | 2 |
| VS5 | require_neq family | 2 | 4 | 37 | 4 | 22 |
| VS6 | close/realloc family | 10 | 5 | 13 | 37 | 21 |
| VS7 | invoke/CpiContext | 15 | 7 | 47 | 6 | 87 |
| VS8 | remaining_accounts | 106 | 9 | 0 | 28 | 26 |
| VS9 | checked/as/saturating | 211 | 122 | 283 | 197 | 163 |
| VS10 | checked_div/floor/try_into | 15 | 8 | 26 | 175 | 21 |
| VS10 | regex a / b * c | 6 | 0 | 0 | 8 | 0 |
| VS11 | rounding tokens | 62 | 0 | 12 | 24 | 0 |
| VS12 | pyth/switchboard | 0 | 0 | 0 | 183 | 0 |
| VS12 | age API + bounds | 0 | 0 | 0 | 32 | 0 |
| VS13 | clock/schedule tokens | 12 | 5 | 3 | 28 | 141 |
| VS14 | token-2022 tokens | 150 | 59 | 107 | 1 | 10 |
| VS14 | dangerous extensions | 34 | 7 | 14 | 0 | 0 |
| VS15 | decimals/pow | 10 | 24 | 8 | 12 | 7 |
| VS16 | require comparison macros | 61 | 12 | 0 | 1 | 153 |
| VS16 | post only | 0 | 0 | 16 | 0 | 0 |
| VS17 | fee/treasury/referr/builder | 1 | 1 | 3 | 45 | 80 |
| VS18 | crank/settle/liquidate | 6 | 0 | 2 | 10 | 7 |
| VS19 | vault/share tokens | 0 | 35 | 0 | 2 | 8 |
| VS20 | payer/authority | 8 | 6 | 27 | 7 | 43 |
| VS21 | sysvar family | 5 | 2 | 148 | 18 | 3 |
| VS22 | defaults/shadowing | 15 | 1 | 25 | 95 | 13 |
| VS23 | init/divisibility | 31 | 7 | 8 | 25 | 22 |
| VS24 | reload family | 137 | 23 | 0 | 0 | 29 |
