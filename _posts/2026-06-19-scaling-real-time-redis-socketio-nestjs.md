---
layout: post
title: "Scaling Real-Time Features with Redis and Socket.io in NestJS"
date: 2026-06-19 12:00:00 +0600
published: false
tags:
  - nestjs
  - redis
  - socket.io
  - backend
description: "How to scale a NestJS WebSocket backend horizontally using the Redis Pub/Sub adapter to manage a distributed real-time signaling ecosystem."
---

Building a real-time signaling server for a single instance is straightforward. You boot up a NestJS application, attach a `@WebSocketGateway()`, and your clients connect. But what happens when your user base grows, and you need to deploy three, five, or fifty backend instances behind a load balancer?

In our recent large-scale real-time project, we faced exactly this challenge. If User A connects to Server Node 1, and User B connects to Server Node 2, how does User A send a WebRTC signaling offer to User B? 

The answer lies in combining the structured elegance of **NestJS**, the real-time reliability of **Socket.io**, and the high-speed Pub/Sub capabilities of **Redis**.

## The Problem: WebSocket Stickiness and State

WebSockets maintain stateful, persistent TCP connections. Unlike a stateless REST API where a load balancer can route any request to any server, a WebSocket client establishes a lasting bond with a specific server instance.

If you don't synchronize these servers, a message broadcasted on Server 1 will only reach the clients connected to Server 1. 

## The Solution: The Redis Adapter

Socket.io provides a solution to this through adapters. An adapter is responsible for routing messages between different server nodes. By using the `@socket.io/redis-adapter`, we effectively turned our fleet of isolated NestJS servers into a unified, distributed signaling plane.

### Implementing in NestJS

NestJS provides a robust WebSocket module. To integrate Redis, we created a custom `IoAdapter`.

```typescript
import { IoAdapter } from '@nestjs/platform-socket.io';
import { ServerOptions } from 'socket.io';
import { createAdapter } from '@socket.io/redis-adapter';
import { createClient } from 'redis';

export class RedisIoAdapter extends IoAdapter {
  private adapterConstructor: ReturnType<typeof createAdapter>;

  async connectToRedis(): Promise<void> {
    const pubClient = createClient({ url: process.env.REDIS_URL });
    const subClient = pubClient.duplicate();

    await Promise.all([pubClient.connect(), subClient.connect()]);

    this.adapterConstructor = createAdapter(pubClient, subClient);
  }

  createIOServer(port: number, options?: ServerOptions): any {
    const server = super.createIOServer(port, options);
    server.adapter(this.adapterConstructor);
    return server;
  }
}
```

Once wired into the main NestJS bootstrap sequence, the magic happens automatically. When you call `server.to(room).emit('event', data)`, the Redis Adapter publishes the event to the Redis cluster. Every other NestJS node receives the event via its Sub client and checks if any connected clients belong to that room. If they do, the event is delivered.

## Beyond Pub/Sub: Redis as a State Store

While the adapter solves message routing, it doesn't solve global state. In a signaling server, you constantly need to answer the question: *"Is User X currently online?"*

Querying the PostgreSQL database for this is far too slow for real-time operations. Instead, we used Redis as an ultra-fast, in-memory state store.

### The Presence Dictionary

When a client connects to a `WebSocketGateway`, the `handleConnection` lifecycle hook fires:

```typescript
@WebSocketGateway({ namespace: '/signaling' })
export class SignalingGateway implements OnGatewayConnection, OnGatewayDisconnect {
  
  async handleConnection(client: Socket) {
    const userId = extractUserIdFromToken(client);
    
    // Store the connection in Redis
    await this.redisService.set(`presence:${userId}`, 'online', 'EX', 300);
    
    // Broadcast to friends that user is online
    client.broadcast.emit('user_status', { userId, status: 'online' });
  }

  async handleDisconnect(client: Socket) {
    const userId = extractUserIdFromToken(client);
    
    // Remove presence from Redis
    await this.redisService.del(`presence:${userId}`);
    
    // Broadcast offline status
    client.broadcast.emit('user_status', { userId, status: 'offline' });
  }
}
```

By storing presence data with an expiration (`EX 300`), we protect against zombie connections in the event of a hard server crash where the `handleDisconnect` hook never fires.

## Dealing with Race Conditions

In a distributed environment, two servers might attempt to establish a session for the same user simultaneously. We utilized Redis distributed locks (using `setnx` or Redlock algorithms) to ensure that critical operations—like generating a unique session ID for a video call room—happened atomically across the entire cluster.

## Conclusion

NestJS provides an incredibly structured way to build WebSocket gateways, but it's Redis that gives it the power to scale. By offloading message routing to the Redis Adapter and moving real-time state out of PostgreSQL and into Redis keys, we built a signaling ecosystem capable of handling immense traffic while maintaining the low-latency guarantees required for WebRTC negotiation.
