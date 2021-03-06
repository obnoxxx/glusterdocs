Object versioning
=================

  Bitrot detection in GlusterFS relies on object (file) checksum (hash) verification,
  also known as "object signature". An object is signed when there are no active
  file desciptors referring to it's inode (i.e., upon last close()). This is just an
  hint for the initiation of hash calculation (and therefore signing). There is
  absolutely no control over when clients can initiate modification operations on
  the object. An object could be under modification while it's hash computation is
  under progress. It would also be in-appropriate to restrict access to such objects
  during the time duration of signing.

  Object versioning is used as a mechanism to identify the staleness of an objects
  signature. The document below does not just list down the version update protocol,
  but goes through various factors that led to its design.

*NOTE:* The word "object" is used to represent a "regular file" (in linux sense) and
      object versions are persisted in extended attributes of the object's inode.
      Signature calculation includes object's data (no metadata as of now).

INDEX
=====
1.  Version updation protocol
2.  Correctness guaraantees
3.  Implementation
4.  Protocol enhancements

1.  Version updation protocol
============================
  There are two types of versions associated with an object:

  a) Ongoing version: This version is incremented on first open() [when
     the in-memory representation of the object (inode) is marked dirty
     and synchronized to disk. When an object is created, a default ongoing
     version of one (1) is assigned. An object lookup() too assigns the
     default version if not present. When a version is initialized upon
     lookup() or creat() FOP, it need to be durable on disk and therefore
     can just be a extended attrbute set with out an expensive fsync()
     syscall.

  b) Signing version: This is the version against which an object is deemed
     to be signed. An objects signature is tied to a particular signed version.
     Since, an object is a candidate for signing upon last release() [last
     close()], signing version is the "ongoing version" at that point of time

  An object's signature is trustable when the version it was signed against
  matches the ongoing version, i.e., if the hash is calculated by hand and
  compared against the object signature, it *should* be a perfect match if
  and only if the versions are equal. On the other hand, the signature is
  considered stale (might or might not match the hash just calculated).

  Initialization of object versions
  ---------------------------------
  An object that existed before the pre versioning days, is assigned the
  default versions upon lookup(). The protocol at this point expects "no"
  durability guarantess of the versions, i.e., extended attribute sets
  need not be followed by an explicit filesystem sync (fsync()). In case
  of a power outage or a crash, versions are re-initialized with defaults
  if found to be non-existant. The signing version is initialized with a
  deafault value of zero (0) and the ongoing version as one (1).
 
  *NOTE:* If an object already has versions on-disk, lookup() just brings
         the versions in memory. In this case both versions may or may
         not match depending on state the object was left in.

  Increment of object versions
  ----------------------------
  During initial versioning, the in-memory representation of the object is
  marked dirty, so that subsequent modification operations on the object
  triggers a versiong synchronization to disk (extended attribute set).
  Moreover, this operation needs to be durable on disk, for the protocol
  to be crash consistent.

  Let's picturize the various version states after subsequent open()s.
  Not all modification operations need to increment the ongoing version,
  only the first operations needs to (subsequent operations are NO-OPs).

  *NOTE:* From here one "[s]" depicts a durable filesystem operation and
           "*" depicts the inode as dirty.


                       lookup()     open()    open()    open()
            ===========================================================

            OV(m):        1*          2         2         2
                      -----------------------------------------
            OV(d):        1           2[s]      2         2
            SV(d):        0           0         0         0


  Let's now picturize the state when an already signed object undergoes
  file operations.

    on-disk state:
    OV(d): 3
    SV(d): 3|<signature>


                       lookup()     open()    open()    open()
            ===========================================================

            OV(m):        3*          4         4         4
                      -----------------------------------------
            OV(d):        3           4[s]      4         4
            SV(d):        3           3         3         3

  Signing process
  ---------------
  As per the above example, when the last open file descriptor is closed,
  signing needs to be performed. The protocol restricts that the signing
  needs to be attached to a version, which in this case is the in-memory
  value of the ongoing version. A release() also marks the inode dirty,
  therefore, the next open() does a durable version synchronization to
  disk.

  [carry forwarding the versions from earlier example]

                       close()     release()  open()   open()
            ===========================================================

            OV(m):        4           4*        5         5
                      -----------------------------------------
            OV(d):        4           4         5[s]      5
            SV(d):        3           3         3         3

  As shown above, a relase() call triggers a signing with signing version
  as OV(m): which in this case is 4. During signing, the object is signed
  with a signature attached to version 4 as shown below (continuing with
  the last open() call from above):

                       open()           sign(4, signature)
            ===========================================================

            OV(m):        5                     5
                      -----------------------------------------
            OV(d):        5                     5
            SV(d):        3               4:<signature>[s]

  A signature comparison at this point of time is un-trustable due to
  version mismatches. This also protects from node crashes and hard
  reboots due to durability guarantee of on-disk version on first
  open().

                       close()     release()  open()
            ===========================================================

            OV(m):        4           4*        5
                      --------------------------------  CRASH
            OV(d):        4           4         5[s]
            SV(d):        3           3         3

  The protocol is immune to signing request after crashes due to
  the version synchronization performed on first open(). Signing
  request for a version lesser than the *current* ongoing version
  can be ignored. It's left upon the implementation to either
  accept or ignore such signing request(s).

  *NOTE:* Inode forget() causes a fresh lookup() to be trigerred.
        Since a forget() call is received when there are no
        active references for an inode, the on-disk version is
        the latest and would be copied in-memory on lookup().
        
2.  Correctness Guarantees
==========================

  Concurrent open()'s
  -------------------
  When an inode is dirty (i.e., the very next operations would try to
  synchronize the version to disk), there can be multiple calls [say,
  open()] that would find the inode state as dirty and try to writeback
  the new version to disk. Also, note that, marking the inode as synced
  and updating the in-memory version is done *after* the new version
  is written on disk. This is done to avoid incorrect version stored
  on-disk in case the version synchronization fails (but the in-memory
  version still holding the updated value).
  Coming back to multiple open() calls on an object, each open() call
  tries to synchronize the new version to disk if the inode is marked
  as dirty. This is safe as each open() would try to synchronize the
  new version (ongoingversion + 1) even if the updation is concurrent.
  The in-memory version is finally updated to reflect the updated
  version and mark the inode non-dirty. Again this is done *only* if
  the inode is dirty, thereby open() calls which updated the on-disk
  version but lost the race to update the in-memory version result
  are NO-OPs.

  on-disk state:
      OV(d): 3
      SV(d): 3|<signature>


                       lookup()     open()    open()'   open()'  open()
            =============================================================

            OV(m):        3*          3*        3*        4      NO-OP
                      --------------------------------------------------
            OV(d):        3           4[s]      4[s]      4        4
            SV(d):        3           3         3         3        3

  open()/release() race
  ---------------------
  This race can cause a release() [on last close()] to pick up the
  ongoing version which was just incremented on fresh open(). This
  leads to signing of the object with the same version as the
  ongoing version, thereby, mismatching signatures when calculated.
  Another point that's worth mentioning here is that the open
  file descriptor is *attached* to it's inode *after* it's done
  version synchronization (and increment). Hence, if a release()
  sneaks in this window, the file desriptor list for the given
  inode is still empty, therefore release() considering it as a
  last close().
  To counter this, the protocol should track the open and release
  counts for file descriptors. A release() should only trigger a
  signing request when the file desccriptor for an inode is empty
  and the numbers of releases match the number of opens. When an
  open() sneaks and increments the ongoing version but the file
  descriptor is still not attached to the inode, open and release
  counts mismatch, hence identifying an open() in progress.

3.  Implementation
==================

Refer to: xlators/feature/bit-rot/src/stub

4.  Protocol enhancements
=========================

  a) Delaying persisting on-disk versions till open()
  b) Lazy version updation (until signing?)
  c) Protocol changes required to handle anonymous file
        descriptors in GlusterFS.
