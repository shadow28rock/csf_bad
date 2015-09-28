#include <stdio.h>
#include <stdlib.h>
#include <sys/types.h>
#include <unistd.h>
#include <pthread.h>
 
// объявляем структуру данных для одного задания
struct producer_consumer_queue_item {
  struct producer_consumer_queue_item *next;
  // здесь идут собственно данные. вы можете поменять этот кусок,
  // использовав структуру, более специфичную для вашей задачи
  void *data;
};
 
// объявляем очередь с дополнительными структурами для синхронизации.
// в этой очереди будут храниться произведённые, но ещё не потреблённые задания.
struct producer_consumer_queue {
  struct producer_consumer_queue_item *head, *tail;
                              // head == tail == 0, если очередь пуста
  pthread_mutex_t lock;       // мьютекс для всех манипуляций с очередью
  pthread_cond_t cond;        // этот cond "сигналим", когда очередь стала НЕ ПУСТОЙ
  int is_alive;               // показывает, не закончила ли очередь свою работу
};
 
// Теперь нам нужны процедуры добавления и извлечения заданий из очереди.
 
void
enqueue (void *data, struct producer_consumer_queue *aq)
{
  volatile struct producer_consumer_queue *q = aq;
  // упакуем задание в новую структуру
  struct producer_consumer_queue_item *p = (typeof(p))malloc(sizeof(*p));
  p->data = data;
  p->next = 0;
 
  // получим "эксклюзивный" доступ к очереди заданий
  pthread_mutex_lock(&aq->lock);
  // ... и добавим новое задание туда:
  if (q->tail)
    q->tail->next = p;
  else {
    q->head = p;
    // очередь была пуста, а теперь нет -- надо разбудить потребителей
    pthread_cond_broadcast(&aq->cond);
  }
  q->tail = p;
  asm volatile ("" : : : "memory");
  // зафиксируем изменения очереди в памяти
 
  // разрешаем доступ всем снова
  pthread_mutex_unlock(&aq->lock);
}
 
void *
dequeue(struct producer_consumer_queue *aq)
{
  volatile struct producer_consumer_queue *q = aq;
  // получаем эксклюзивный доступ к очереди:
  pthread_mutex_lock(&aq->lock);
 
  while (!q->head && q->is_alive) {
    // очередь пуста, делать нечего, ждем...
    pthread_cond_wait(&aq->cond, &aq->lock);
    // wait разрешает доступ другим на время ожидания
  }
 
  // запоминаем текущий элемент или 0, если очередь умерла
  struct producer_consumer_queue_item *p = q->head;
 
  if (p)
  {
    // и удаляем его из очереди
    q->head = q->head->next;
    if (!q->head)
      q->tail = q->head;
    asm volatile ("" : : : "memory");
    // зафиксируем изменения очереди в памяти
  }
 
  // возвращаем эксклюзивный доступ другим участникам
  pthread_mutex_unlock(&aq->lock);
 
  // отдаём данные
  void *data = p ? p->data : 0; // 0 означает, что данных больше не будет
  // согласно 7.20.3.2/2, можно не проверять на 0
  free(p);
  return data;
}
 
struct producer_consumer_queue *
producer_consumer_queue_create()
{
  struct producer_consumer_queue *q = (typeof(q))malloc(sizeof(*q));
  q->head = q->tail = 0;
  q->is_alive = 1;
  pthread_mutex_init(&q->lock, 0);
  pthread_cond_init(&q->cond, 0);
 
  return q;
}
 
// И процедура для закрытия очереди:
 
void
producer_consumer_queue_stop(struct producer_consumer_queue *aq)
{
  volatile struct producer_consumer_queue *q = aq;
  // для обращения к разделяемым переменным необходим эксклюзивный доступ
  pthread_mutex_lock(&aq->lock);
  q->is_alive = 0;
  asm volatile ("" : : : "memory");
  // зафиксируем изменения очереди в памяти
  pthread_cond_broadcast(&aq->cond); // !!!!! avp
  pthread_mutex_unlock(&aq->lock);
}
 
// это поток-потребитель
void *
consumer_thread (void *arg)
{
  struct producer_consumer_queue *q = (typeof(q))arg;
 
  for (;;) {
    void *data = dequeue(q);
    // это сигнал, что очередь окончена
    if (!data)
      break; // значит, пора закрывать поток
 
    char *str = (char *)data;
    // тут наша обработка данных
    printf ("consuming: %s\n", str);
    sleep(2); // 2000 заменил на 2 avp
    printf ("consumed: %s\n", str);
    free(str);
  }
  return 0;
}
 
int
main ()
{
  pthread_t consumer_threads[2];
 
  void *res = 0;
 
  char *in = NULL;
  size_t sz;
  int l;
 
  // создадим очередь:
  struct producer_consumer_queue *q = producer_consumer_queue_create(); //add struct
 
  // и потоки-«потребители» (изм. consumer на consumer_thread)
  pthread_create(&consumer_threads[0], 0, consumer_thread, (void *)q);
  pthread_create(&consumer_threads[1], 0, consumer_thread, (void *)q);
 
  // главный цикл
  // получаем данные с клавиатуры: (изм. убрал l, добавил puts())
  while ((puts("Enter"), getline(&in, &sz, stdin)) > 0) {
    enqueue(in, q);
    in = NULL;
  }
 
  producer_consumer_queue_stop(q);
  puts("Fin2");
 
  if (pthread_join(consumer_threads[0], &res) ||
      pthread_join(consumer_threads[1], &res))
    perror("join");
 
  return (long)res;
}
