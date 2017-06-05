  ---
layout: post
title: ReentrantLock源码阅读
category: 学习
tags: jdk concurrent
keywords:
description:
---


ReentrantLock 中实现的aqs同步器


            abstract static class Sync extends AbstractQueuedSynchronizer {
                    private static final long serialVersionUID = -5179523762034025860L;

                    /**
                     * Performs {@link Lock#lock}. The main reason for subclassing
                     * is to allow fast path for nonfair version.
                     */
                    abstract void lock();

                    /**
                     * Performs non-fair tryLock.  tryAcquire is
                     * implemented in subclasses, but both need nonfair
                     * try for trylock method.
                     */
                    final boolean nonfairTryAcquire(int acquires) {
                        final Thread current = Thread.currentThread();
                        int c = getState();
                        if (c == 0) {
                            if (compareAndSetState(0, acquires)) {
                                setExclusiveOwnerThread(current);
                                return true;
                            }
                        }
                        else if (current == getExclusiveOwnerThread()) {
                            int nextc = c + acquires;
                            if (nextc < 0) // overflow
                                throw new Error("Maximum lock count exceeded");
                            setState(nextc);
                            return true;
                        }
                        return false;
                    }

                    protected final boolean tryRelease(int releases) {
                        int c = getState() - releases;
                        if (Thread.currentThread() != getExclusiveOwnerThread())
                            throw new IllegalMonitorStateException();
                        boolean free = false;
                        if (c == 0) {
                            free = true;
                            setExclusiveOwnerThread(null);
                        }
                        setState(c);
                        return free;
                    }

                    protected final boolean isHeldExclusively() {
                        // While we must in general read state before owner,
                        // we don't need to do so to check if current thread is owner
                        return getExclusiveOwnerThread() == Thread.currentThread();
                    }

                    final ConditionObject newCondition() {
                        return new ConditionObject();
                    }

                    // Methods relayed from outer class

                    final Thread getOwner() {
                        return getState() == 0 ? null : getExclusiveOwnerThread();
                    }

                    final int getHoldCount() {
                        return isHeldExclusively() ? getState() : 0;
                    }

                    final boolean isLocked() {
                        return getState() != 0;
                    }

                    /**
                     * Reconstitutes this lock instance from a stream.
                     * @param s the stream
                     */
                    private void readObject(java.io.ObjectInputStream s)
                        throws java.io.IOException, ClassNotFoundException {
                        s.defaultReadObject();
                        setState(0); // reset to unlocked state
                    }
                }


### 公平锁

        static final class FairSync extends Sync {
                private static final long serialVersionUID = -3000897897090466540L;

                final void lock() {
                    acquire(1);
                }

                /**
                 * Fair version of tryAcquire.  Don't grant access unless
                 * recursive call or no waiters or is first.
                 */
                protected final boolean tryAcquire(int acquires) {
                    final Thread current = Thread.currentThread();
                    int c = getState();
                    if (c == 0) {
                        if (!hasQueuedPredecessors() &&
                            compareAndSetState(0, acquires)) {
                            setExclusiveOwnerThread(current);
                            return true;
                        }
                    }
                    else if (current == getExclusiveOwnerThread()) {
                        int nextc = c + acquires;
                        if (nextc < 0)
                            throw new Error("Maximum lock count exceeded");
                        setState(nextc);
                        return true;
                    }
                    return false;
                }
            }

### 非公平锁

        static final class NonfairSync extends Sync {
                private static final long serialVersionUID = 7316153563782823691L;

                /**
                 * Performs lock.  Try immediate barge, backing up to normal
                 * acquire on failure.
                 */
                final void lock() {
                    if (compareAndSetState(0, 1))
                        setExclusiveOwnerThread(Thread.currentThread());
                    else
                        acquire(1);
                }

                protected final boolean tryAcquire(int acquires) {
                    return nonfairTryAcquire(acquires);
                }
            }                
