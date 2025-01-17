---
title: "Introduction to Parallel Computing"
teaching: 25
exercises: 20
questions:
- "How does MPI work?"
objectives:
- "Introduce parallel computing."
- "Use `mpicc`/`mpifort` and `mpirun`."
keypoints:
- "Many problems can be distributed across several processors and solved faster."
- "`mpirun` runs copies of the program."
- "The copies are separated MPI rank."
---

## Parallel Computing

In many fields of science we need more computational power than a single core on a single processor can provide.
In essence, parallel computing means using more than one computer (or more than one core) to solve a problem faster.

Naively, using more CPUs (or cores) means that one can solve a problem much faster, in time scales that make sense for research projects or study programs.
However, how efficiently a problem can be solved in parallel depends on how the problem is divided among available CPUs (or cores), and the data and memory needs of the algorithm. This determines how much the cores need to communicate with each other while working together.
The number of cores we can efficiently use also depends on the problem.
You probably have two or four cores on your laptop, and many problems can be run very efficiently on all of those cores.
As a researcher, you probably have access to a High-Performance Computing (HPC) system with thousands or hundreds of thousands of cores.
To use them all efficiently would be challenging in almost any field.

Also, if not careful, sometimes running in parallel can give a wrong result. Consider the sum, 1 - 1 + 1 - 1 + 1 ... Depending on how this summing is performed on multiple CPUs (or cores), the final answer is different. In practice, since there are always round-off errors in numerical calculations and the order of numerical calculations in parallel computing can be different, the result from running on a serial program on one CPU can be different from the result from running a parallel program on multiple CPUs.

During this course you will learn to design parallel algorithms and write parallel programs using the MPI library.
MPI stands for Message Passing Interface, and is a low level, minimal and extremely flexible set of commands for communicating between copies of a program.

## Using MPI

> ## Running with mpirun
>
> Let's get our hands dirty from the start and make sure MPI is installed. Open a bash command line and run the command:
> ~~~
> echo Hello World!
> ~~~
>{: .language-bash}
>
> Now use `mpirun` to run the same command:
> ~~~
> mpirun -n 4 echo Hello World!
> ~~~
> {: .language-bash}
>
> What did mpirun do?
>
> If `mpirun` is not found, try `mpiexec` instead.
> This is another common name for the command.
>
{: .challenge}

> ## MPI on HPC
>
> HPC clusters typically have more than one version of MPI available, so you may need
> to tell it which one you want to use before it will give you access to it.
>
> Typically, a command like `module load mpi` will give you access to an MPI library,
> but ask a helper or consult your HPC facility's documentation if you're not sure how
> to use MPI on a particular cluster.
{: .callout}

> ## Note for Cygwin users
>
> If you use Cygwin, you may notice an error message saying
> ```There are not enough slots available in the system```.
> By default, Cygwin will prevent you from running more copies
> than you have cores to run on. Add `--oversubscribe` after `mpirun`
> to run more copies than you have cores.
>
{: .callout}

Just running a program with `mpirun` starts several copies of it.
The number of copies is decided by the `-n` parameter.
In the example above, the program does not know it was started by `mpirun`
and each copy just works as if they were the only one.

For the copies to work together, they need to know about their role in the computation.
This usually also requires knowing the total number of tasks running at the same time.

To achieve this, the program needs to call the
`MPI_Init` function.
This will set up the environment for MPI, and assign a number (called the _rank_) to each process.
At the end, each process should also cleanup by calling `MPI_Finalize`:

~~~
int MPI_Init(&argc, &argv);
int MPI_Finalize();
~~~
{: .language-c .show-c}

Both `MPI_Init` and `MPI_Finalize` return an integer.
This describes errors that may happen in the function.
Usually we will return the value of `MPI_Finalize` from the main function
{: .show-c}

~~~
integer ierr
call MPI_Init(ierr);
call MPI_Finalize(ierr);
~~~
{: .language-fortran .show-fortran}

Both function take the parameter `ierr`,
which is overwritten by the number
of any error encountered, or by 0 if there is no error.
{: .show-fortran}

Between these two statements, you can find out the rank of the copy using the `MPI_Comm_rank` function:

~~~
int rank;
int MPI_Comm_rank(MPI_COMM_WORLD, &rank);
~~~
{: .language-c .show-c}

~~~
integer rank
call MPI_Comm_rank(MPI_COMM_WORLD,rank,ierr)
~~~
{: .language-fortran .show-fortran}


Here's a more complete example:

~~~
 #include <stdio.h>
 #include <mpi.h>

 int main(int argc, char** argv) {
   int rank;

   // First call MPI_Init
   MPI_Init(&argc, &argv);
   // Get my rank
   MPI_Comm_rank(MPI_COMM_WORLD, &rank);

   printf("My rank number is = %d\n", rank);

   // Call finalize at the end
   return MPI_Finalize();
 }
~~~
{: .language-c .show-c}

~~~
program hello

    implicit none
    include "mpif.h"

    integer rank, ierr

    call MPI_Init(ierr)
    call MPI_Comm_rank(MPI_COMM_WORLD,rank,ierr)
    write(6,*) "My rank number is ", rank
    call MPI_Finalize(ierr)

end
~~~
{: .source .language-fortran .show-fortran}

> ## Fortran Standard
>
>Fortran examples, exercises and solutions in this workshop will conform to the
>Fortran 2008 standard.
>The standard can be specified to the compiler in two ways:
>* using the `.F08` file extension.
>* adding `-std=f2008` on the command line.
>
>Fortran 2008 should be readable to those familiar with an earlier standard.
>
>The Fortran 2008 standard contains the `use` statement, which is a more native
>way of referring to modules and can replace the `include` statement borrowed from C.
>You can try replacing `include "mpi.h"` with `use mpi`, which should come before the
>`implicit none` statement.
>However, as some versions of the MPI library do not support Fortran modules, we will
>stick with the `include` statement in the examples.
>
{: .callout .show-fortran}


If you try to compile these examples with your usual C or Fortran compiler, then you
will receive an error, since these do not link to the MPI libraries that provide the
new functions that we have introduced. Since the set of compiler flags that would be
necessary for this tends to be quite long, MPI libraries such as OpenMPI provide
compiler wrappers, which set up the correct environment before invoking your compiler
of choice. The wrapper for C compilers is usually called `mpicc`, while the wrapper
for Fortran is usually called `mpifort`. These can be called in exactly the same way
as your usual C compiler, e.g. `gcc` and `gfortran`, respectively. Any options not
recognised by the MPI wrapper are passed through to the non-MPI compiler.

> ## Compile
>
> Compile the above code with
>`mpicc`{: .show-c}
>`mpifort`{: .show-fortran}
>, and run it with `mpirun`.
>
{: .challenge}

> ## Communicators
>
> The `MPI_COMM_WORLD` parameter is an `MPI communicator`, which was created by `MPI_Init`.
> It labels the set of cores we are working with, called ranks, and provides a context for
> communications between them. You can also create your own communicators, which contain
> a subset of the MPI ranks. Because of this, most MPI functions require you to specify
> the communicator you want them to operate on---for this lesson, this will always be `MPI_COMM_WORLD`.
{: .callout}

Notice that we are still only coding for a single copy of the program. This is always true when using
MPI. Each copy runs the same code and only differs by its rank number.
The best way to think about writing MPI code is to focus on what a single rank
needs to be doing.
When all ranks are doing their job, the algorithm will work correctly.

Usually the rank will need to know how many other ranks there are. You can find this out using the `MPI_Comm_size` function.

~~~
int n_ranks;
int MPI_Comm_size(MPI_COMM_WORLD, &n_ranks);
~~~
{: .language-c .show-c}

~~~
integer n_ranks
MPI_Comm_size(MPI_COMM_WORLD, n_ranks);
~~~
{: .language-fortran .show-fortran}


## Hello World!
> ## Hello World!
>
> Use the MPI library to write a "Hello World" program.
> Each copy of the program, or rank, should print "Hello World!" followed by its rank number.
> They should also print the total number of ranks.
>
>> ## Solution
>>
>> ~~~
>> #include <stdio.h>
>> #include <mpi.h>
>>
>> int main(int argc, char** argv) {
>>   int rank, n_ranks;
>>
>>   // First call MPI_Init
>>   MPI_Init(&argc, &argv);
>>   // Get my rank and the number of ranks
>>   MPI_Comm_rank(MPI_COMM_WORLD, &rank);
>>   MPI_Comm_size(MPI_COMM_WORLD, &n_ranks);
>>
>>   printf("Hello World! I'm rank %d\n", rank);
>>   printf("total no. of ranks = %d\n", n_ranks);
>>
>>   // Call finalize at the end
>>   return MPI_Finalize();
>> }
>> ~~~
>>{: .source .language-c}
>{: .solution .show-c}
>
>> ## Solution
>> ~~~
>>program hello
>>
>>     implicit none
>>     include "mpif.h"
>>     
>>     integer rank, n_ranks, ierr
>>
>>     call MPI_Init(ierr)
>>     call MPI_Comm_rank(MPI_COMM_WORLD,rank,ierr)
>>     call MPI_Comm_size(MPI_COMM_WORLD,n_ranks,ierr)
>>     write(6,*) "Hello World! I'm rank ", rank
>>     write(6,*) "total no. of ranks = ", n_ranks
>>     call MPI_Finalize(ierr)
>>end
>> ~~~
>>{: .source .language-fortran}
>{: .solution .show-fortran}
>
{: .challenge}



>## Parallel loop
>
> Modify the following code to split the iterations of the loop among processes.
> Each iteration of the loop should be run by only one rank.
> Each rank should also have more or less the same amount of work.
>
> ~~~
> #include <stdio.h>
>
> int main(int argc, char** argv) {
>   int numbers = 10;
>
>   for( int i=0; i<numbers; i++ ) {
>     printf("I'm printing the number %d.\n", i);
>   }
>
> }
> ~~~
> {: .source .language-c .show-c}
>
> ~~~
>program printnumbers
>    implicit none
>    
>    integer numbers, number
>    numbers = 10
>
>    do number = 0, numbers - 1
>         write(6,*) "I'm printing the number", number
>    end do
>end
> ~~~
> {: .source .language-fortran .show-fortran}
>
>
>
>> ## Solution
>> ~~~
>>#include <stdio.h>
>>#include <math.h>
>>#include <mpi.h>
>>
>>int main(int argc, char** argv) {
>>  int rank, n_ranks, numbers_per_rank;
>>  int my_first, my_last;
>>  int numbers = 10;
>>
>>  // First call MPI_Init
>>  MPI_Init(&argc, &argv);
>>  // Get my rank and the number of ranks
>>  MPI_Comm_rank(MPI_COMM_WORLD,&rank);
>>  MPI_Comm_size(MPI_COMM_WORLD,&n_ranks);
>>
>>  // Calculate the number of iterations for each rank
>>  numbers_per_rank = floor(numbers/n_ranks);
>>  if( numbers%n_ranks > 0 ){
>>     // Add 1 in case the number of ranks doesn't divide the number of numbers
>>     numbers_per_rank += 1;
>>  }
>>
>>  // Figure out the first and the last iteration for this rank
>>  my_first = rank * numbers_per_rank;
>>  my_last = my_first + numbers_per_rank;
>>
>>  // Run only the part of the loop this rank needs to run
>>  // The if statement makes sure we don't go over
>>  for( int i=my_first; i<my_last; i++ ) {
>>    if( i < numbers ) {
>>      printf("I'm rank %d and I'm printing the number %d.\n", rank, i);
>>    }
>>  }
>>
>>  // Call finalize at the end
>>  return MPI_Finalize();
>>}
>> ~~~
>> {: .source .language-c}
>>
>{: .solution .show-c}
>
>
>> ## Solution
>> ~~~
>>program print_numbers
>>
>>    implicit none
>>    include "mpif.h"
>>
>>    integer numbers, number
>>    integer rank, n_ranks, ierr
>>    integer numbers_per_rank, my_first, my_last
>>    numbers = 10
>>
>>    ! Call MPI_Init
>>    call MPI_Init(ierr)
>>
>>    ! Get my rank and the total number of ranks
>>    call MPI_Comm_rank(MPI_COMM_WORLD,rank,ierr)
>>    call MPI_Comm_size(MPI_COMM_WORLD,n_ranks,ierr)
>>
>>    ! Calculate the number of iterations for each rank
>>    numbers_per_rank = numbers/n_ranks
>>    if (MOD(numbers, n_ranks) > 0) then
>>         ! add 1 in case the number of ranks doesn't divide the number of numbers
>>         numbers_per_rank = numbers_per_rank + 1
>>    end if
>>
>>    ! Figure out the first and the last iteration for this rank
>>    my_first = rank * numbers_per_rank
>>    my_last = my_first + numbers_per_rank
>>
>>    ! Run only the necessary part of the loop, making sure we don't go over
>>    do number = my_first, my_last - 1
>>         if (number < numbers) then
>>              write(6,*) "I'm rank", rank, " and I'm printing the number", number
>>         end if
>>    end do
>>
>>
>>    call MPI_Finalize(ierr)
>>end
>> ~~~
>>{: .source .language-fortran}
>>
>{: .solution .show-fortran}
>
>
{: .challenge }

In lesson 3 you will learn how to communicate between the ranks. From there, you can in principle write general parallel programs.
The trick is in designing a working, fast and efficient parallel algorithm for you problem.


{% include links.md %}
