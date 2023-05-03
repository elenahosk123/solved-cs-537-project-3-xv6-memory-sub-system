Download Link: https://assignmentchef.com/product/solved-cs-537-project-3-xv6-memory-sub-system
<br>
<h2 id="fq">F&amp;Q</h2>

Please refer the <a href="https://piazza.com/class/k4bkdfbqqziw5?cid=745">F&amp;Q</a> post on the piazza.

<h2 id="updates">Updates</h2>

<ol>

 <li>For the <code>rand.h</code> file given in the spec<pre><code> Original: #define	XV6_RAND_MAX = 2147483647 Updated:  #define	XV6_RAND_MAX 2147483647</code></pre></li>

 <li>If applicable, submit <code>partner.login</code> and <code>SLIP_DAYS</code> file to <code>/&lt;cs-login&gt;/p3a/</code> directory</li>

</ol>

<h2 id="teaming-up"><em>Teaming up!</em></h2>

For this project, you have the option to work with a partner. Read more details in <a href="http://pages.cs.wisc.edu/~shivaram/cs537-sp20/p3.html#submitting-your-implementation">Submitting Your Implementation</a> and <a href="http://pages.cs.wisc.edu/~shivaram/cs537-sp20/p3.html#collaboration">Collaboration</a>.

<h2 id="administrivia">Administrivia</h2>

<ul>

 <li><strong>Due Date</strong> by Mar 5, 2020 at 10:00 PM</li>

 <li><strong>You have two submissions for this assignment</strong>.</li>

 <li>Questions: We will be using Piazza for all questions.</li>

 <li>This project is to be done on the <a href="https://csl.cs.wisc.edu/services/instructional-facilities">lab machines</a>, so you can learn more about programming in C on a typical UNIX-based platform (Linux).</li>

 <li>Please take the <strong>quiz on Canvas</strong> that will check if you have read the spec closely. NOTE: The quiz does not work in groups and will have to be taken individually on Canvas.</li>

 <li>Your group (i.e., you and your partner) will have <em>2</em> slip days for the rest of the semester. If you are changing partners please talk to the instructor about it.</li>

</ul>

<h2 id="objectives">Objectives</h2>

<ul>

 <li>To understand how to add the memory protection for pages in xv6 system.</li>

 <li>To understand how the process of page allocation works and add two basic page allocator implementations.</li>

</ul>

<h2 id="overview">Overview</h2>

In the first part of project, you’ll be adding read only access to some pages in xv6, to protect the process from accidentally modifying it.

You will be implementing 2 system calls:

<ul>

 <li><code>int mprotect(void *addr, int len)</code> to add read-only protection to pages.</li>

 <li><code>int munprotect(void *addr, int len)</code> to give read-write permission to pages back.</li>

</ul>

In the second part, you will implement two basic page allocation schemes. The first scheme allocates frames alternatively to prevent malicious writes. The second scheme allocates frames randomly. You also need to implement a random number generator for second scheme. You will implement a system call <code>int dump_allocated(int *frames, int numframes) </code>which dumps information about the allocated frames.

<h3 id="reading">Reading:</h3>

<ul>

 <li><a href="https://pdos.csail.mit.edu/6.828/2018/xv6/book-rev11.pdf">Chapter 2</a> in xv6 book.</li>

</ul>

<h2 id="part-1-read-only-code">Part 1: <strong>Read-only</strong> Code</h2>

In most operating systems, code is marked read-only instead of read-write. However, in xv6 this is not the case, so a buggy program could accidentally overwrite its own text. Try it and see!

In this portion of the xv6 project, you’ll change the protection bits of parts of the page table to be read-only, thus preventing such over-writes, and also be able to change them back.

To do this, you’ll be adding two system calls: <code>int mprotect(void *addr, int len)</code> and <code>int munprotect(void *addr, int len)</code>.

Calling <code>mprotect()</code> should change the protection bits of the page range starting at <code>addr</code> and of <code>len</code> pages to be read only. Thus, the program could still read the pages in this range after <code>mprotect()</code> finishes, but a write to this region should cause a trap (and thus kill the process). The <code>munprotect()</code> call does the opposite: sets the region back to both readable and writeable.

Also required: the page protections should be inherited on fork(). Thus, if a process has mprotected some of its pages, when the process calls fork, the OS should copy those protections to the child process.

There are some failure cases to consider: if <code>addr</code> is not page aligned, or <code>addr</code> points to a region that is not currently a part of the address space, or <code>len</code> is less than or equal to zero, return -1 and do not change anything. Otherwise, return 0 upon success.

Hint: after changing a page-table entry, you need to make sure the hardware knows of the change. On 32-bit x86, this is readily accomplished by updating the <code>CR3</code> register (what we generically call the <em>page-table base register</em> in class). When the hardware sees that you overwrote <code>CR3</code> (even with the same value), it guarantees that your PTE updates will be used upon subsequent accesses. The <code>lcr3()</code> function will help you in this pursuit.

<h2 id="page-allocation">Page-allocation</h2>

As you know, the abstraction of an address space per process is a great way to provide isolation (i.e., protection) across processes. However, even with this abstraction, attackers can still try to modify or access portions of memory that they do not have permission for.

For example, in a <a href="https://en.wikipedia.org/wiki/Row_hammer">Rowhammer</a> attack, a malicious process can repeatedly write to certain addresses in order to cause bit flips in DRAM in nearby (physical) addresses that the process does not have permission to write directly. In this part, you’ll implement one (not very sophisticated) ways to alleviate the impact of a malicious process: allocating pages from different processes so that they are not physically adjacent.

One of the advantages of page-based systems is that a virtual page can be allocated in any free physical page (or frame); all physical pages are supposed to be equivalent. However, not paying attention to frame allocation could allow a malicious process to repeatedly write to addresses at the edge of one of its pages, which due to the low-level properties of modern DRAM, could then modify the values at the edge of the next physical page which happens to belong to a different process.

<ul>

 <li>The current physical memory allocator in xv6 uses a simple free list. You can find the routines for allocating physical memory and managing this simple linked list in <code>kalloc.c</code>.   By looking at <code>kinit()</code> you can see how the free list is initially created to include all physical pages across a range and you can see that kalloc() simply returns the first page on that free list when a frame is needed.</li>

 <li>Note that the free list starts with the highest numbered free frame and begins sorted in reverse order.</li>

 <li>You will modify these allocation routines to implement your new allocation scheme.</li>

 <li>Note that kalloc() should fail (i.e., return NULL) when it cannot find a suitable page.</li>

</ul>

There are a different levels of sophistication for page allocation to reduce the physical memory wastage. For this project however, we ask you to write two very basic implementations. In both these implementations, you will be modifying the <code>kalloc()</code> method.

<h3 id="part-2-alternate-allocation">Part 2: Alternate allocation</h3>

In this implementation, you keep a free frame between every allocated frame in the system. For example, if kalloc() sees 3 frame allocations for process a, then 2 frame allocations for process b, and then 2 for process a again, it will end up allocating the pages of physical memory to processes as follows (F represents a free page): aFaFaFbFbFaFa. This approach satisfies our security requirements, but uses twice as much memory as the original xv6 approach.

<strong>The changes for alternate allocation as well as Part 1 should be submitted in p3a subdirectory</strong>. We will run test cases for these two parts in your p3a subdirectory.

<h3 id="part-3-random-allocation">Part 3: Random Allocation</h3>

In this implementation, the allocator assigns random entries in the free list. For this purpose, you will need to implement a pseudo random number generator, which gives a random number each time. We have provided a header file <code>rand.h</code> below.

<pre><code>#ifndef _RAND_H#define _RAND_H#define	XV6_RAND_MAX  2147483647/*     Return a random integer between 0 and XV6_RAND_MAX inclusive.     NOTE: If xv6_rand is called before any calls to xv6_srand have been made, the same     sequence shall be generated as when xv6_srand is first called with a seed value of 1.*/int xv6_rand (void);/*     The xv6_srand function uses the argument as a seed for a new sequence of pseudo-random numbers to be returned by subsequent calls to rand.    If xv6_srand is then called with the same seed value, the sequence of pseudo-random numbers shall be repeated.    If xv6_rand is called before any calls to xv6_srand have been made, the same sequence shall be generated as when xv6_srand is first called with a seed value of 1.  */void xv6_srand (unsigned int seed);#endif // _RAND_H</code></pre>

You have to implement the two functions <code>xv6_rand()</code> and <code>xv6_srand()</code> in <code>rand.c</code>. The <code>xv6_rand()</code> method generates a random number on each call and <code>xv6_srand()</code> method is used to set a seed. We recommend using the <a href="https://en.wikipedia.org/wiki/Xorshift">Xorshift family</a> of random number generators, although you are welcome to use other approaches.

Each time in your kalloc implementation, the allocator should call the xv6_rand() method to generate a random number <code>x</code>. Then, it takes its remainder with the current free list size to get <code>y</code>, and returns the <code>yth</code> frame from start in the free list. For example: if the remainder is 0, the first frame in free list is allocated, when remainder is 1 second frame is allocated .. and so on. Also, don’t forget to remove the allocated frame from the free list!

<h4 id="details">Details</h4>

<ul>

 <li>You have to keep both <code>rand.h</code> and <code>rand.c</code> in the kernel subdirectory.</li>

 <li>You should not change the initialisation of free list portion in the <code>kinit()</code> method. Our test cases assume standard free list initilisation. You can however use variables to maintain size of free list and history of allocated frames.</li>

 <li>The <code>xv6_rand()</code> method generates a random number based on some seed. When the seed is not specifically set using <code>xv6_srand()</code> method, it assumes seed of 1. However, when a seed is set, the next call to <code>xv6_rand()</code> should give the first random number corresponding to that seed.</li>

 <li>For ex:, suppose for some <code>xv6_rand()</code> method implementation, the following code gives output as below:</li>

</ul>

<pre><code>xv6_srand(2);printf(" %d
", xv6_rand()); //outputs 13438xv6_srand(5);printf(" %d
", xv6_rand()); //outputs 64829</code></pre>

Then for the following code also, output should be independent of previous seed.

<pre><code>xv6_rand();xv6_srand(5);printf(" %d
", xv6_rand()); //should output 64829</code></pre>

<strong>The changes for random allocation should be submitted in p3b subdirectory</strong>. We will run test cases for random allocation on submissions in p3b subdirectory.

<h3 id="dump-allocated-pages">Dump allocated pages</h3>

You also have to implement a system call to dump the pages which have been allocated. This will help you debug your kernel and us to test your code.

<pre><code>int dump_allocated(int *frames, int numframes) </code></pre>

<ul>

 <li><strong>frames</strong>: a pointer to an allocated array of integers that will be filled in by the kernel with a list of all the frame numbers that are currently allocated</li>

 <li><strong>numframes</strong>: The previous <em>numframes</em> allocated frames whose information we are asking for.</li>

 <li><strong>Note</strong>: You only have to return the frame numbers of the frames which have been allocated, not the ones left empty. For example, your freelist has frames with address as follows: <code>(23, 19, 16, 15, 13, 8, 5, 3)</code> . It then allocates four frames with address <code>8, 19, 5, 15</code> in order. The freelist should now become <code>(23, 16, 13, 3)</code>. And a call to <code>dump_allocted(frames, 3)</code>, should have <code>frames</code> pointing to array <code>(15, 5, 19)</code>.</li>

 <li>Return -1 on error (e.g., something wrong with input parameters, or numframes is greater than number of allocated pages till now) and 0 on success.</li>

 <li><strong>This method should work correctly for both the allocation schemes above and be present in both submissions.</strong></li>

</ul>

<h3 id="submitting-your-implementation">Submitting Your Implementation</h3>

<ol>

 <li>Please create two subdirectories ~cs537-1/handin/LOGIN/p3a/src and ~cs537-1/handin/LOGIN/p3b/src.</li>

</ol>

<pre><code>mkdir -p ~cs537-1/handin/LOGIN/p3a/srcmkdir -p ~cs537-1/handin/LOGIN/p3b/src </code></pre>

To submit your solution, copy all of the xv6 files and directories with your changes for alternate allocation and Part 1 into <code>~cs537-1/handin/&lt;cs-login&gt;/p3a/src</code>, and changes for random allocation into <code>~cs537-1/handin/&lt;cs-login&gt;/p3b/src</code>. One way to do this is to navigate to your solution’s working directory and execute the following commands:

<pre><code>cd &lt;your p3a working directory&gt;cp -r . ~cs537-1/handin/&lt;cs-login&gt;/p3a/src/cd &lt;your p3b working directory&gt;cp -r . ~cs537-1/handin/&lt;cs-login&gt;/p3b/src/</code></pre>

If you choose to work in pairs, only one of you needs to submit the source code. But, both of you need to submit an additional <code>partner.login</code> file inside <code>/&lt;cs-login&gt;/p3a/&gt;</code>, which contains one line that has the CS login of your partner (and nothing else). If there is no such file, we are assuming you are working alone. If both of you turn in your code, we will just randomly choose one to grade. Consider the following when you submit your project:

<ul>

 <li>If you use any slip days you have to create a <code>SLIP_DAYS</code> file in the <code>/&lt;cs-login&gt;/p3a/</code> directory otherwise we use the submission on the due date.</li>

 <li>Do not remove the <code>README</code> file in the main directory</li>

 <li>Do not change the permissions</li>

 <li>Do not remove the other files that you are not modifying</li>

 <li>Do not modify or remove the Makefile</li>

 <li>Your files should be directly copied to <code>~cs537-1/handin/&lt;cs-login&gt;/p3a/src</code> directory. Having subdirectories in <code>&lt;cs-login&gt;/p3a/src</code> like <code>&lt;cs-login&gt;/p3a/src/xv6-sp20</code> or … is <strong>not acceptable</strong>.</li>

</ul>


