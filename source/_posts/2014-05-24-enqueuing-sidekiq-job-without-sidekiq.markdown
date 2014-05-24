---
layout: post
title: "Enqueuing Sidekiq jobs without Sidekiq"
date: 2014-05-25 14:00:00 +0200
comments: true
categories:
---

Let's say you have a running Sidekiq system and for whatever reason you need to enqueue
jobs from another not ruby-based environment or system. It's actually quite simple as you
can directly push your jobs into Redis without the Sidekiq gem.

First, let's create a very simple worker and enqueue it:

``` ruby
class HardWorker
  include Sidekiq::Worker

  def perform(name)
    puts "Hey, I'm a worker named #{name}"
  end
end

HardWorker.perform_async("foo")
# => "d34299988658f23c62c178da"
```

Sidekiq gives us back a unique ID for that specific job. Let's have a look inside Redis:

```
redis 127.0.0.1:6379> type "queues"
set

redis 127.0.0.1:6379> type "queue:default"
list

redis 127.0.0.1:6379> smembers queues
1) "default"
```

Now for simple jobs (as opposed to *scheduled* jobs), Sidekiq use two different keys: a list and a set. The set called `queues` by default only store the queues names. The list named `queue:nameofthequeue` actually store the job informations.  Let's have a closer look:

```
redis 127.0.0.1:6379> LRANGE queue:default 0 1
1) "{\"retry\":true,\"queue\":\"default\",\"class\":\"HardWorker\",\"args\":[\"foo\"],\"jid\":\"d34299988658f23c62c178da\",\"enqueued_at\":1400959039.450082}"
```

A-Ha! So a job is simply a hash serialized in JSON. Those are the required keys:

* `retry` (boolean): tells Sidekiq whether to retry or not a failed job
* `queue` (string): self-explanatory!
* `class` (string): the class of the worker that will be instantiated by Sidekiq
* `args` (array): the arguments that will passed to the worker's contructor
* `jid` (string): the unique ID of the job
* `enqueued_at` (float): the timestamp when the job was enqueued

Pretty simple, huh? So, to enqueue a job yourself you have to:

* Generate a unique ID.
* Serialize the payload using JSON.
* Add the name of the queue to the `queues` set (using `SADD`).
* Push the payload to the `queue:myqueue` list (using `LPUSH`).

Pretty simple, eh? Now you might be wondering, what about scheduled job? Well it's not that much more complicated! First let's push a scheduled job:

```
irb> HardWorker.perform_in(10.minutes, "foo")
  => "672512fcf9ba85078d73bd77"
```

Then have a look at what's inside Redis:

```
redis 127.0.0.1:6379> keys *
1) "schedule"

redis 127.0.0.1:6379> type "schedule"
zset

redis 127.0.0.1:6379> zrange schedule 0 1
1) "{\"retry\":true,\"queue\":\"default\",\"class\":\"HardWorker\",\"args\":[\"foo\"],\"jid\":\"672512fcf9ba85078d73bd77\",\"enqueued_at\":1400959918.936842}"
```

At first sight, there's not much difference besides the fact that the job is stored in a sorted set (`zset`) instead of a list. But it's in a sorted set, that means it must have score:

```
redis 127.0.0.1:6379> zscore "schedule" "{\"retry\":true,\"queue\":\"default\",\"class\":\"HardWorker
  "1400959928.9367521"
```

The score is actually the time at which the job is supposed to be executed! This allow Sidekiq to use `ZRANGEBYSCORE` to simply pop the jobs that should be executed [^1]. Now if you want to enqueue a scheduled job, you just have to add it to the `schedule` sorted set using `ZADD`!

[^1]: [Here's where the magic happens in the Sidekiq code](https://github.com/mperham/sidekiq/blob/master/lib/sidekiq/scheduled.rb#L35)

And really, that's all there is to it! As long as you know the class name of the worker and the arguments it take, you can enqueue jobs from any programming language. Or even directly inside Redis if you wish so!

For the sake of completion, here's a naive implementation in Go:

``` go Enqueue a Sidekiq job in Go https://gist.github.com/DuoSRX/8f3290d3d93e0054fe35 Gist
package main

import (
  "crypto/rand"
  "encoding/hex"
  "encoding/json"
  "fmt"
  "github.com/garyburd/redigo/redis"
  "io"
  "time"
)

type Job struct {
  JID        string   `json:"jid"`
  Retry      bool     `json:"retry"`
  Queue      string   `json:"queue"`
  Class      string   `json:"class"`
  Args       []string `json:"args"`
  EnqueuedAt int64    `json:"enqueued_at"`
}

func randomHex(n int) string {
  id := make([]byte, n)
  io.ReadFull(rand.Reader, id)
  return hex.EncodeToString(id)
}

func (job *Job) Enqueue() string {
  conn, err := redis.Dial("tcp", ":6379")
  if err != nil {
    panic(err)
  }

  encoded, _ := json.Marshal(job)

  conn.Send("SADD", "queues", job.Queue)
  conn.Send("LPUSH", "queue:"+job.Queue, string(encoded))
  conn.Flush()
  conn.Close()

  return job.JID
}

func NewJob(class, queue string, args []string) *Job {
  job := &Job{
    JID:        randomHex(12),
    Retry:      false,
    Queue:      queue,
    Class:      class,
    Args:       args,
    EnqueuedAt: time.Now().Unix(),
  }

  return job
}

func main() {
  job := NewJob("HardWorker", "default", []string{"foo"})
  fmt.Println(job.Enqueue())
}

```
