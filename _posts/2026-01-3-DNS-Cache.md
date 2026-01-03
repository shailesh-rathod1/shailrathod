---
layout: post
title: "DNS Cache in C"
date: 2026-01-03 15:00:00 +0530
categories: [Blog]
description: "DNS Cache in C , 5M insertion in ~15-20 Sec"

---

### Without optimization ###
```c
#include<stdio.h>
#include<stdlib.h>
#include<string.h>
#include<time.h>


#include<stdbool.h>

#define MAX_DOMAIN_LEN 256
#define MAX_IP_LEN 16


typedef struct dns_record {
    char domain[MAX_DOMAIN_LEN];
    char ip[MAX_IP_LEN];
    time_t timestamp;

    long ttl;

    struct dns_record *next;
    struct dns_record *prev;
}dns_record_t;

typedef struct hash_bucket {
    dns_record_t *head;
    struct hash_bucket * next;

}hash_bucket_t; 

typedef struct dns_cache {
    hash_bucket_t **buckets;
    size_t num_buckets;
    size_t size;    
    int capacity;

    dns_record_t *lru_head;
    dns_record_t *lru_tail;
}dns_cache_t;

static dns_cache_t * cache = NULL;

static unsigned long hash_string (const char *str, int num_buckets) {
    unsigned long hash = 5381;
    int c;
    while ((c = *str++)) {
        hash = ((hash << 5) + hash ) + c;
    }

    return hash % num_buckets;
}

static void move_to_head(dns_record_t * record) {
    if(cache->lru_head == record) {
        return;
    }

    if (record->prev) {
        record->prev->next = record->next;
    }

    if(record->next) {
        record->next->prev  = record->prev;
    }

    if(cache->lru_tail == record) cache->lru_tail = record->prev;

    record->next = cache->lru_head;
    record->prev  = NULL;

    if(cache->lru_head) cache->lru_head->prev = record;
    cache->lru_head = record;

    if(cache->lru_tail == NULL) cache->lru_tail = record;

}

//Removee recorddd frooom LRUUUUU LIIIISTTTTT

static void remove_from_lru(dns_record_t * record) {
    if(record->prev) {
        record->prev->next = record->next;
    }else
        cache->lru_head = record->next;

    if(record->next) {
        record->next->prev = record->prev;
    }else  
        cache->lru_tail = record->prev;
}

//Remove record froom hashhhhh tabbbbbleeee chaaaainnnnn 
static void remove_from_hash(dns_record_t * record, unsigned long bucket_idx) {
    hash_bucket_t * curr = cache->buckets[bucket_idx];
	getchar();

    hash_bucket_t * prev = NULL;

    while(curr) {
        if (curr->head == record) {
            hash_bucket_t * temp = curr;
            if(prev == NULL) {
                cache->buckets[bucket_idx] = cache->buckets[bucket_idx]->next;
            } else {
                prev->next = curr->next;
            }
            return;
        }

        prev = curr;
        curr = curr->next;
    }
}

static void evict_lru() {
    if(cache->lru_tail == NULL)
        return;

    dns_record_t * record = cache->lru_tail;

    unsigned long bucket_idx = hash_string(record->domain,cache->num_buckets);
    
    remove_from_hash(record,bucket_idx);
    remove_from_lru(record);
    free(record);
    cache->size--;
}

bool dns_cache_init(int capacity) {
    if (capacity <= 0)
        return false;

    cache = (dns_cache_t *) malloc(sizeof(dns_cache_t));
    if(cache == NULL)
        return false;

    cache->capacity = capacity;
    cache->size = 0;
    cache->num_buckets = capacity * 2;
    cache->lru_head = NULL;
    cache->lru_tail = NULL;

    cache->buckets = (hash_bucket_t **) calloc(cache->num_buckets, sizeof(hash_bucket_t *));
    if (cache->buckets == NULL) {
        free(cache);
        cache = NULL;
        return false;
    }

    return true; 
}

bool dns_cache_lookup(const char * domain , char *ip_buffer, size_t buffer_size) {
    if (!cache || !domain || !ip_buffer)
        return false;

    unsigned long bucket_idx  = hash_string(domain, cache->num_buckets);
	hash_bucket_t * bucket = cache->buckets[bucket_idx];
	
	while(bucket) {
		
		if(bucket->head && strcmp(bucket->head->domain, domain) == 0) {
			// Check if entry is expired
			time_t ct = time(NULL);
			if (ct - bucket->head->timestamp > bucket->head->ttl) {
				// Entry expired, remove it
				remove_from_hash(bucket->head, bucket_idx);
				remove_from_lru(bucket->head);
				free(bucket->head);
				bucket->head = NULL;
				cache->size--;
				return false;
			}
			
			//valid entry found 
			strncpy(ip_buffer, bucket->head->ip, buffer_size - 1);
			ip_buffer[buffer_size - 1] = '\0';
			
			//Move to head (MRU)
			move_to_head(bucket->head);
			
			return true;
			
		}
		
		bucket = bucket->next;
	}
	
	return false;
}

bool dns_cache_insert(const char * domain, const char *ip, long ttl_seconds) {
	
	if(!cache || !domain || !ip || ttl_seconds <= 0)
		return false;
	
	unsigned long bucket_idx = hash_string(domain, cache->num_buckets);
	hash_bucket_t * entry = cache->buckets[bucket_idx];
	
	while(entry) {
		if(entry->head && strcmp(entry->head->domain, domain) == 0) {
			//Update the existing entry
			strncpy(entry->head->ip, ip, MAX_IP_LEN - 1);
			entry->head->ip[MAX_IP_LEN - 1] = '\0';
			entry->head->timestamp = time(NULL);
			entry->head->ttl = ttl_seconds;
			
			//Move to head
			move_to_head(entry->head);
			
			return true;
		}
		
		entry = entry->next;
	}
	
	
	if(cache->size >= cache->capacity)
		evict_lru();
	
	
	dns_record_t * new_record = (dns_record_t *) malloc(sizeof(dns_record_t));
	
	if(!new_record)
		return false;
	
	strncpy(new_record->domain, domain, MAX_DOMAIN_LEN - 1);
	new_record->domain[MAX_DOMAIN_LEN - 1] = '\0';
	strncpy(new_record->ip, ip , MAX_IP_LEN - 1);
	new_record->ip[MAX_IP_LEN - 1] = '\0';
	new_record->timestamp = time(NULL);
	new_record->ttl = ttl_seconds;
	new_record->prev = NULL;
	new_record->next = NULL;
	
	hash_bucket_t * new_bucket = (hash_bucket_t *) malloc(sizeof(hash_bucket_t));
	
	new_bucket->head = new_record;
	new_bucket->next = cache->buckets[bucket_idx];
	cache->buckets[bucket_idx] = new_bucket;
	
	
	new_record->next = cache->lru_head;
	if(cache->lru_head)
			cache->lru_head->prev = new_record;
	cache->lru_head = new_record;
	
	if(cache->lru_tail == NULL) {
		cache->lru_tail = new_record;
	}
	
	cache->size++;
	
	return true;
}

void dns_cache_destroy() {
	if (!cache)
		return;
	
	dns_record_t * curr  = cache->lru_head;
	
	while(curr) {
		dns_record_t * next = curr->next;
		free(curr);
		curr = next;
	}
	
	hash_bucket_t *bucket = *cache->buckets;
	
	while(bucket) {
		hash_bucket_t * next = bucket->next;
		free(bucket);
		bucket = next;
	}
	
	free(cache->buckets);
	free(cache);
	cache = NULL;
	
	
}

void dns_cache_print() {
	if(!cache)
		return;
	
	printf("DNS cache (size : %d/%d):\n", cache->size, cache->capacity);
	printf("LRU Order (head = most recent): \n");
	
	dns_record_t * curr = cache->lru_head;
	
	int idx = 0;
	
	while(curr) {
		time_t ct = time(NULL);
		long age = ct - curr->timestamp;
		printf(" [%d] %s -> %s (age : %lds, ttl: %lds, %s)\n",
				idx++, curr->domain, curr->ip, age, curr->ttl,
				age > curr->ttl ? "Expired" : "Valid");
		curr = curr->next;
	}
}







int main() {
    printf("DNS CACEH IMPLLLLLLLLLLLL");
	
	if(!dns_cache_init(3)) {
		printf("Failed to initialize cache\n");
		return 1;
	}
	
	char ip_buffer[MAX_IP_LEN];
	
	printf("Test 1 : Inserting entries \n");
	dns_cache_insert("google.com", "142.250.185.46", 300);
	dns_cache_insert("github.com", "140.82.121.4", 300);
	dns_cache_insert("stackoverflow.com", "151.101.1.69", 300);
	dns_cache_print();
	printf("\n");
	
	if(dns_cache_lookup("google.com", ip_buffer, MAX_IP_LEN)) {
		printf("Found : google.com -> %s\n", ip_buffer);
	}else	
		printf("Not found\n");
	dns_cache_print();
	printf("\n");
	
	dns_cache_insert("amazon.com","179.32.103.205", 300);
	dns_cache_print();
	printf("\n");
	
	if(dns_cache_lookup("github.com", ip_buffer, MAX_IP_LEN)) {
		printf("Found : github.com -> %s\n", ip_buffer);
	}else	
		printf("Not found\n");
	
	dns_cache_insert("google.com", "142.250.185.786", 300);
	dns_cache_print();
	printf("\n");
	
    dns_cache_insert("test.com", "1.2.3.4", 2);
	//dns_cache_print();
	printf("waiting for 3 seconds \n");
	sleep(3);
	
	if(dns_cache_lookup("test.com", ip_buffer, MAX_IP_LEN)) {
		printf("Found : test.com -> %s\n", ip_buffer);
	}else	
		printf("Not found\n");
	
	
	
	
    return 0;
}
```

### Optimized 5M insertion in ~15-20 sec ###

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <time.h>
#include <stdbool.h>

#define MAX_DOMAIN_LEN 256
#define MAX_IP_LEN 16

/* ===================== DATA STRUCTURES ===================== */

typedef struct dns_record {
    char domain[MAX_DOMAIN_LEN];
    char ip[MAX_IP_LEN];
    time_t expiry;

    struct dns_record *hash_next;
    struct dns_record *lru_prev;
    struct dns_record *lru_next;
} dns_record_t;

typedef struct dns_cache {
    dns_record_t **buckets;
    size_t num_buckets;

    size_t size;
    size_t capacity;

    dns_record_t *lru_head;
    dns_record_t *lru_tail;

    /* MEMORY POOL */
    dns_record_t *pool;
    size_t pool_index;
} dns_cache_t;

static dns_cache_t *cache = NULL;

/* ===================== UTIL ===================== */

static size_t next_power_of_two(size_t n) {
    size_t p = 1;
    while (p < n) p <<= 1;
    return p;
}

static unsigned long hash_string(const char *str) {
    unsigned long hash = 5381;
    int c;
    while ((c = *str++))
        hash = ((hash << 5) + hash) + c;
    return hash;
}

/* ===================== LRU HELPERS ===================== */

static void lru_remove(dns_record_t *rec) {
    if (rec->lru_prev)
        rec->lru_prev->lru_next = rec->lru_next;
    else
        cache->lru_head = rec->lru_next;

    if (rec->lru_next)
        rec->lru_next->lru_prev = rec->lru_prev;
    else
        cache->lru_tail = rec->lru_prev;
}

static void lru_insert_head(dns_record_t *rec) {
    rec->lru_prev = NULL;
    rec->lru_next = cache->lru_head;

    if (cache->lru_head)
        cache->lru_head->lru_prev = rec;

    cache->lru_head = rec;

    if (!cache->lru_tail)
        cache->lru_tail = rec;
}

static void lru_move_to_head(dns_record_t *rec) {
    if (cache->lru_head == rec)
        return;
    lru_remove(rec);
    lru_insert_head(rec);
}

/* ===================== HASH REMOVE ===================== */

static void hash_remove(const char *domain, size_t idx) {
    dns_record_t *cur = cache->buckets[idx];
    dns_record_t *prev = NULL;

    while (cur) {
        if (strcmp(cur->domain, domain) == 0) {
            if (prev)
                prev->hash_next = cur->hash_next;
            else
                cache->buckets[idx] = cur->hash_next;
            return;
        }
        prev = cur;
        cur = cur->hash_next;
    }
}

/* ===================== EVICT ===================== */

static void evict_lru(void) {
    if (!cache->lru_tail)
        return;

    dns_record_t *victim = cache->lru_tail;
    size_t idx = hash_string(victim->domain) & (cache->num_buckets - 1);

    lru_remove(victim);
    hash_remove(victim->domain, idx);

    /* NOTE: no free() – memory stays in pool */
    cache->size--;
}

/* ===================== API ===================== */

bool dns_cache_init(size_t capacity) {
    if (capacity == 0)
        return false;

    cache = calloc(1, sizeof(dns_cache_t));
    if (!cache)
        return false;

    cache->capacity = capacity;
    cache->num_buckets = next_power_of_two((capacity * 4) / 3);

    cache->buckets = malloc(cache->num_buckets * sizeof(dns_record_t *));
    if (!cache->buckets)
        return false;

    memset(cache->buckets, 0,
           cache->num_buckets * sizeof(dns_record_t *));


    /* MEMORY POOL */
    cache->pool = malloc(sizeof(dns_record_t) * capacity);
    if (!cache->pool)
        return false;

    cache->pool_index = 0;
    return true;
}

bool dns_cache_lookup(const char *domain, char *ip_buf, size_t buf_len) {
    if (!cache || !domain || !ip_buf)
        return false;

    size_t idx = hash_string(domain) & (cache->num_buckets - 1);
    dns_record_t *cur = cache->buckets[idx];
    time_t now = time(NULL);

    while (cur) {
        if (strcmp(cur->domain, domain) == 0) {
            if (cur->expiry <= now) {
                lru_remove(cur);
                hash_remove(domain, idx);
                cache->size--;
                return false;
            }

            strncpy(ip_buf, cur->ip, buf_len - 1);
            ip_buf[buf_len - 1] = '\0';
            lru_move_to_head(cur);
            return true;
        }
        cur = cur->hash_next;
    }
    return false;
}

bool dns_cache_insert(const char *domain, const char *ip, long ttl) {
    if (!cache || !domain || !ip || ttl <= 0)
        return false;

    size_t idx = hash_string(domain) & (cache->num_buckets - 1);
    dns_record_t *cur = cache->buckets[idx];
    time_t now = time(NULL);

    while (cur) {
        if (strcmp(cur->domain, domain) == 0) {
            strncpy(cur->ip, ip, MAX_IP_LEN - 1);
            cur->ip[MAX_IP_LEN - 1] = '\0';
            cur->expiry = now + ttl;
            lru_move_to_head(cur);
            return true;
        }
        cur = cur->hash_next;
    }

    if (cache->size >= cache->capacity)
        evict_lru();

    if (cache->pool_index >= cache->capacity)
        return false;

    dns_record_t *rec = &cache->pool[cache->pool_index++];

    strncpy(rec->domain, domain, MAX_DOMAIN_LEN - 1);
    rec->domain[MAX_DOMAIN_LEN - 1] = '\0';
    strncpy(rec->ip, ip, MAX_IP_LEN - 1);
    rec->ip[MAX_IP_LEN - 1] = '\0';
    rec->expiry = now + ttl;

    rec->hash_next = cache->buckets[idx];
    cache->buckets[idx] = rec;

    rec->lru_prev = rec->lru_next = NULL;
    lru_insert_head(rec);

    cache->size++;
    return true;
}

void dns_cache_destroy(void) {
    if (!cache)
        return;

    free(cache->pool);
    free(cache->buckets);
    free(cache);
    cache = NULL;
}

/* ===================== TEST ===================== */

int main(void) {
    dns_cache_init(5000000);

    char ip[16];
    clock_t start = clock();

    for (int i = 0; i < 5000000; i++) {
       if (i % 100000 == 0) {
       // printf("progress: %d\n", i);
        fflush(stdout);
        }
        char d[32], a[16];
        snprintf(d, sizeof(d), "g%d", i);
        snprintf(a, sizeof(a), "1.%d", i);
        dns_cache_insert(d, a, 300);
        dns_cache_lookup(d, ip, sizeof(ip));
    }
     fflush(stdout);

    clock_t end = clock();
    printf("Time: %.3f ms\n",
           1000.0 * (end - start) / CLOCKS_PER_SEC);
          

    dns_cache_destroy();
    return 0;
}

```
