#include <pthread.h>
#include <stdlib.h>
#include "csesem.h"
#include "pcq.h"

//man pthread_cond_wait
//man pthread_mutex_lock
//man sem_overview
//man pthread_create
//man pthread_join
//man pthread_exit
//man pthread_cancel
struct PCQueue {
    int slots; //maximum number of slots
    void **queue;
    int front; //head of queue
    int rear; //tail of queue
    CSE_Semaphore s;
    CSE_Semaphore scount; //counts available slots
    CSE_Semaphore items; //counts number of items
};

PCQueue pcq_create(int slots) {
    if(slots <= 0){
        return NULL;
    }
    PCQueue pcq;
    pcq = calloc(1, sizeof(*pcq));
    pcq->queue = calloc(slots, sizeof(void *));
    pcq->slots = slots;
    pcq->front = 0;
    pcq->rear = 0;
    pcq->s = csesem_create(1);
    pcq->scount = csesem_create(slots);
    pcq->items = csesem_create(0);

    return pcq;
}
//void csesem_post(CSE_Semaphore sem) { //v
//void csesem_wait(CSE_Semaphore sem) { //p
void pcq_insert(PCQueue pcq, void *data) {
    csesem_wait(pcq->scount);
    csesem_wait(pcq->s);
    pcq->queue[(++pcq->rear)%(pcq->slots)] = data;
    csesem_post(pcq->s);
    csesem_post(pcq->items);
}

void *pcq_retrieve(PCQueue pcq) {
    void* item;
    csesem_wait(pcq->items);
    pcq->iw = (pcq->iw) + 1;
    csesem_wait(pcq->s);
    pcq->sw = (pcq->sw) + 1;

    item = pcq->queue[(++pcq->front)%(pcq->slots)];
    
    csesem_post(pcq->s);
    pcq->sw = (pcq->sw) - 1;
    csesem_post(pcq->scount);
    pcq->scw = (pcq->scw) - 1;
    return item;
    //BlessRNG
}
------------------------------------------------
struct CSE_Semaphore {
    int data;
    pthread_mutex_t m;
    pthread_cond_t cv;   
};

CSE_Semaphore csesem_create(int count) {
    if(count < 0){
        return NULL;
    }
    CSE_Semaphore sem = calloc(1, sizeof(struct CSE_Semaphore));
    sem->data = count;
    pthread_mutex_init(&(sem->m), NULL);
    pthread_cond_init((&(sem->cv)), NULL);
    return sem;
}

void csesem_post(CSE_Semaphore sem) { //v
    pthread_mutex_lock(&(sem->m));
    sem->data = (sem->data) + 1;
    pthread_cond_signal(&(sem->cv));
    pthread_mutex_unlock(&(sem->m));
}

void csesem_wait(CSE_Semaphore sem) { //p
    pthread_mutex_lock(&(sem->m));
    if((sem->data)>0){
        sem->data = (sem->data) - 1;
    }
    //else if((sem->data)<0){
        //sem->data = 0;
        //}
    else{
        while((sem->data) == 0){
            pthread_cond_wait(&(sem->cv), &(sem->m));
        }
        sem->data = (sem->data) - 1;
    }
    pthread_mutex_unlock(&(sem->m));
}

void csesem_destroy(CSE_Semaphore sem) {
    pthread_mutex_destroy(&(sem->m));
    pthread_cond_destroy(&(sem->cv));
    free(sem);
}
