---
title: 常见的字符串Hash算法
---

## 常见hash算法的碰撞概率统计

|            | 10万     | 50万      | 100万     | 500万      | 1000万     | 一亿         | 1000万次的平均执行时间 | 一亿次的平均执行时间 | 一亿次的平均长度 |
|------------|---------|----------|----------|-----------|-----------|------------|---------------|------------|----------|
| BKDRHash   | 0.00002 | 0.000112 | 0.000251 | 0.0011894 | 0.0023321 | 0.0229439  | 0.0064134     | 0.00968998 | 9        |
| APHash     | 0       | 0.000052 | 0.000122 | 0.0005794 | 0.0011712 | 0.01155826 | 0.0061518     | 0.01088634 | 10       |
| DJBHash    | 0.00001 | 0.00011  | 0.000204 | 0.0011782 | 0.0023154 | 0.02294341 | 0.0064836     | 0.01098645 | 9        |
| JSHash     | 0       | 0.000188 | 0.00032  | 0.001464  | 0.0029323 | 0.02876141 | 0.0063464     | 0.00904354 | 9        |
| RSHash     | 0.00001 | 0.000122 | 0.000245 | 0.001154  | 0.00233   | 0.02290588 | 0.0063627     | 0.01168532 | 9        |
| SDBMHash   | 0.00002 | 0.000132 | 0.000235 | 0.001175  | 0.0023435 | 0.02294529 | 0.0064155     | 0.01201398 | 9        |
| PJWHash    | 0.00312 | 0.015032 | 0.029957 | 0.1386394 | 0.251465  | 0.83290663 | 0.0067549     | 0.00601705 | 8        |
| ELFHash    | 0.00096 | 0.005584 | 0.011239 | 0.0539746 | 0.1028391 | 0.52002744 | 0.0060441     | 0.00704438 | 9        |
| MurmurHash | 0       | 0        | 0        | 0         | 0         | 0          | 0.0066868     | 0.01194736 | 19       |
| CityHash   | 0       | 0        | 0        | 0         | 0         | 0          | 0.0066179     | 0.01129171 | 19       |
| FNVHash    | 0.00005 | 0.000186 | 0.000349 | 0.0016688 | 0.0033469 | 0.03279751 | 0.0061614     | 0.01018707 | 9        |
| crc64      | 0       | 0        | 0        | 0         | 0         | 0          | 0.0064459     | 0.01242473 | 19       |

1.BKDR hash function
```c
unsigned int bkdr_hash(const char *str)
{
    unsigned int seed = 131; // the magic number, 31, 131, 1313, 13131, etc.. orz..
    unsigned int hash = 0;
    unsigned char *p = (unsigned char *)str;
    while (*p)
        hash = hash * seed + (*p++);
    return hash;
}
```

2.AP hash function
```c
unsigned int ap_hash(char *str)
{
    unsigned int hash = 0;
    int i;
    for (i=0; *str; i++)
        if ((i & 1) == 0)
            hash ^= ((hash << 7) ^ (*str++) ^ (hash >> 3));
        else
            hash ^= (~((hash << 11) ^ (*str++) ^ (hash >> 5)));
    return (hash & 0x7FFFFFFF);
}
```

3.DJB hash function
```c
unsigned long hash_djbx33a(const char *str, size_t len)
{
    unsigned long hash = 0U;
    for(size_t i = 0;i < len; ++i) {
        hash = hash * 33 + (unsigned long)str[i];
        /* or, hash = ((hash << 5) + hash) + (unsigned long)str[i]; 
         * where, hash * 33 = ((hash << 5) + hash)
         */
    }

    return hash;
}

long long djb2(char s[])
{
    long long hash = 5381; /* init value */
    int i = 0;
    while (s[i] != '\0')
    {
        hash = ((hash << 5) + hash) + s[i];
        i++;
    }
    return hash;
}
```

4.JS hash function
```c
unsigned int js_hash(char*str)
{
    unsigned int hash = 1315423911 ;
    while(*str)
    {
        hash ^=((hash <<5 ) + (*str++) + (hash >>2 ));
    }
    return hash;
}
```

5.RS hash function
```c
unsigned int RSHash( char * str)
{
    unsigned int b = 378551;
    unsigned int a = 63689;
    unsigned int hash = 0;
    while(*str)
    {
        hash = hash * a + (*str++);
        a *= b;
    }
    return (hash & 0x7FFFFFFF);
}
```

6.SDBM hash function
```c
static unsigned long sdbm(unsigned char *str)
{
    unsigned long hash = 0;
    int c;
    while (c = *str++)
        hash = c + (hash << 6) + (hash << 16) - hash;
    return hash;
}
```
sdbm_hash.c
```c
/* Licensed to the Apache Software Foundation (ASF) under one or more
 * contributor license agreements.  See the NOTICE file distributed with
 * this work for additional information regarding copyright ownership.
 * The ASF licenses this file to You under the Apache License, Version 2.0
 * (the "License"); you may not use this file except in compliance with
 * the License.  You may obtain a copy of the License at
 *
 *     http://www.apache.org/licenses/LICENSE-2.0
 *
 * Unless required by applicable law or agreed to in writing, software
 * distributed under the License is distributed on an "AS IS" BASIS,
 * WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
 * See the License for the specific language governing permissions and
 * limitations under the License.
 */

/*
 * sdbm - ndbm work-alike hashed database library
 * based on Per-Aake Larson's Dynamic Hashing algorithms. BIT 18 (1978).
 * author: oz@nexus.yorku.ca
 * status: ex-public domain. keep it that way.
 *
 * hashing routine
 */

#include "apr_sdbm.h"
#include "sdbm_private.h"

/*
 * polynomial conversion ignoring overflows
 * [this seems to work remarkably well, in fact better
 * then the ndbm hash function. Replace at your own risk]
 * use: 65599  nice.
 *      65587  even better. 
 */
long sdbm_hash(const char *str, int len)
{
    register unsigned long n = 0;

#define DUFF/* go ahead and use the loop-unrolled version */
#ifdef DUFF

#define HASHCn = *str++ + 65599 * n

    if (len > 0) {
        register int loop = (len + 8 - 1) >> 3;

        switch(len & (8 - 1)) {
        case 0:do { 
               HASHC;case 7:HASHC;
        case 6:HASHC; case 5:HASHC;
        case 4:HASHC; case 3:HASHC;
        case 2:HASHC;case 1:HASHC;
            } while (--loop);
        }
    }
#else
    while (len--)
        n = *str++ + 65599 * n;
#endif
    return n;
}
```

7.PJWHash hash function
```c
/* 
 * A generic hash function HashPJW better than ElfHash point, 
 * but depending on the context.
 */
#include <limits.h>
#define BITS_IN_int     ( sizeof(int) * CHAR_BIT )
#define THREE_QUARTERS  ((int) ((BITS_IN_int * 3) / 4))
#define ONE_EIGHTH      ((int) (BITS_IN_int / 8))
#define HIGH_BITS       ( ~((unsigned int)(~0) >> ONE_EIGHTH ))

unsigned int HashPJW ( const char * datum )
{
    unsigned int hash_value, i;
    for ( hash_value = 0; *datum; ++datum )
    {
        hash_value = ( hash_value << ONE_EIGHTH ) + *datum;
        if (( i = hash_value & HIGH_BITS ) != 0 )
            hash_value = ( hash_value ^ ( i >> THREE_QUARTERS )) & ~HIGH_BITS;
    }
    return ( hash_value );
}
```

8.ELFHash hash function
```c
/*
 *    This function hash the input string 'name' using the ELF hash
 *    function for strings.
 */
static unsigned int hash(char* name)
{
    unsigned int h = 0;
    unsigned int g;

    while(*name) {
        h = (h<<4) + *name++;
        if ((g = (h & 0xf0000000)))
            h ^= g>>24;
        h &=~ g;
    }
    return h;
}
```

9.Murmur hash function
```c
uint32_t murmur3_32(const uint8_t* key, size_t len, uint32_t seed) {
    uint32_t h = seed;
    if (len > 3) {
        const uint32_t* key_x4 = (const uint32_t*) key;
        size_t i = len >> 2;
        do {
            uint32_t k = *key_x4++;
            k *= 0xcc9e2d51;
            k = (k << 15) | (k >> 17);
            k *= 0x1b873593;
            h ^= k;
            h = (h << 13) | (h >> 19);
            h = (h * 5) + 0xe6546b64;
        } while (--i);
        key = (const uint8_t*) key_x4;
    }
    if (len & 3) {
        size_t i = len & 3;
        uint32_t k = 0;
        key = &key[i - 1];
        do {
        k <<= 8;
        k |= *key--;
        } while (--i);
        k *= 0xcc9e2d51;
        k = (k << 15) | (k >> 17);
        k *= 0x1b873593;
        h ^= k;
    }
    h ^= len;
    h ^= h >> 16;
    h *= 0x85ebca6b;
    h ^= h >> 13;
    h *= 0xc2b2ae35;
    h ^= h >> 16;
    return h;
}
```

10.City hash function
```c
/* 
 * CityHash 的主要优点是大部分步骤包含了至少两步独立的数学运算
 * 代码较同类流行算法复杂
 */
```

11.FNVHash
FNV-1 hash
```code
hash = FNV_offset_basis
for each byte_of_data to be hashed
    hash = hash × FNV_prime
    hash = hash XOR byte_of_data
return hash
```
FNV-1a hash
```code
hash = FNV_offset_basis
for each byte_of_data to be hashed
    hash = hash XOR byte_of_data
    hash = hash × FNV_prime
return hash
```

## 其它的比较
| Hash函数   | 数据1 | 数据2 | 数据3  | 数据4 | 数据1得分 | 数据2得分 | 数据3得分 | 数据4得分 | 平均分   |
|----------|-----|-----|------|-----|-------|-------|-------|-------|-------|
| BKDRHash | 2   | 0   | 4774 | 481 | 96.55 | 100   | 90.95 | 82.05 | 92.64 |
| APHash   | 2   | 3   | 4754 | 493 | 96.55 | 88.46 | 100   | 51.28 | 86.28 |
| DJBHash  | 2   | 2   | 4975 | 474 | 96.55 | 92.31 | 0     | 100   | 83.43 |
| JSHash   | 1   | 4   | 4761 | 506 | 100   | 84.62 | 96.83 | 17.95 | 81.94 |
| RSHash   | 1   | 0   | 4861 | 505 | 100   | 100   | 51.58 | 20.51 | 75.96 |
| SDBMHash | 3   | 2   | 4849 | 504 | 93.1  | 92.31 | 57.01 | 23.08 | 72.41 |
| PJWHash  | 30  | 26  | 4878 | 513 | 0     | 0     | 43.89 | 0     | 21.95 |
| ELFHash  | 30  | 26  | 4878 | 513 | 0     | 0     | 43.89 | 0     | 21.95 |

其中数据1为100000个字母和数字组成的随机串哈希冲突个数。数据2为100000个有意义的英文句子哈希冲突个数。数据3为数据1的哈希值与1000003(大素数)求模后存储到线性表中冲突的个数。数据4为数据1的哈希值与10000019(更大素数)求模后存储到线性表中冲突的个数。

经过比较，得出以上平均得分。平均数为平方平均数。可以发现，BKDRHash无论是在实际效果还是编码实现中，效果都是最突出的。APHash也是较为优秀的算法。DJBHash,JSHash,RSHash与SDBMHash各有千秋。PJWHash与ELFHash效果最差，但得分相似，其算法本质是相似的。

在信息修竞赛中，要本着易于编码调试的原则，个人认为BKDRHash是最适合记忆和使用的。