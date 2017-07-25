# ece391-os

 Minimalist operating system built from scratch. The terminal uses the x86 infrastructure and implements many 
 Linux-based features. The operating systems supports paging, multiple terminals, mutli-threading, scheduling,
 and the ability to load and execute C programs! The project is developed in x86 assembly and low level C language. 
 
## Making sense of the code

Open the ```student-distrib``` file to look at the raw source code for the operating system. 

## Paging

We used paging to set map virtual memory to physical memory. 

Here is an example of initialization of paging with video memory chunks supported in multiple terminals.

```
entry = VIDEO;
    entry |= RWON;
    page_table[VIDEO >> 12] = entry;
    page_table[(VIDEO >> 12) + 1] = BACKUP0 | RWON;
    page_table[(VIDEO >> 12) + 2] = BACKUP1 | RWON;
page_table[(VIDEO >> 12) + 3] = BACKUP2 | RWON;
```
We also had to enable certain control registers to activate paging. Here is the ```enable_paging``` function,
which took care of this.

```
void enable_paging()
{
    //cr3 holds page directory address (physical address)
    //oring cr4 allows to enable virtual interrupts in protected mode
    //oring cr0 enables pagin, enables protected mode
    asm volatile(
                "movl %0, %%eax \n \
                movl %%eax, %%cr3 \n \
                movl %%cr4, %%eax \n \
                orl $0x00000010, %%eax \n \
                movl %%eax, %%cr4 \n \
                movl %%cr0, %%eax \n \
                orl $0x80000001, %%eax \n \
                movl %%eax, %%cr0"
                :
                :"r"(page_directory)
                :"%eax"
                );
}
```

## Scheduling

We implemented Round-Robin scheduling to support running multiple processes at the same time. 

To time the scheduling, we used a PIT (programmable interval timer) and switched programs contexts based on this timer.

Once a PIT interrupt is detected, this code determines which process is next.

```
 //if next process is not in the active terminal write to backup video memory
    if((schedule_arr[curr]->proc_id)/4 != curr_terminal){
        int term_to_write = (curr_pcb->proc_id)/4;
        page_directory[VIDMAP_PAGE] = (uint32_t)page_table_vid | URWON;
        page_table_vid[0] = sched_vid_backups[term_to_write] | URWON;
        page_table[VIDEO >> 12] = sched_vid_backups[term_to_write] | RWON;
    }
    //next process is in current terminal so write to active memory
    else{
      page_directory[VIDMAP_PAGE] = (uint32_t)page_table_vid | URWON;
      page_table_vid[0] = VIDEO | URWON;
      page_table[VIDEO >> 12] = VIDEO | URWON;
}
```
And next, we perform a context switch to the next process.

```
//context switching
    tss.esp0 = (uint32_t)(schedule_arr[curr])+STACK_SIZE4;
    tss.ss0 = KERNEL_DS;

    //save old esp/ebp
    if(schedule_arr[last]){
        asm (
          "movl %%esp, %0"
          :"=r"(schedule_arr[last]->sched_esp)
        );
        asm (
          "movl %%ebp, %0"
          :"=r"(schedule_arr[last]->sched_ebp)
        );
    }

    //restore new esp/ebp
    asm (
      "movl %0, %%esp"
      :
      :"r"(schedule_arr[curr]->sched_esp)
    );
    asm (
      "movl %0, %%ebp"
      :
      :"r"(schedule_arr[curr]->sched_ebp)
    );

    //set paging to new process
    page_directory[32] = mem_locs[schedule_arr[curr]->idx] | SURWON;
```
Then we need to alter video memory.

 ```
  memcpy((void*)VIDEO,(const void*)vid_backups[curr_terminal],PG_SIZE);
  memcpy((void*)line_char_buffer,(const void*)buf_backups[curr_terminal],BUFFER_MAX_INDEX+1);
  set_buf_idx(buff_idx_backups[curr_terminal]);
  placeCursor(xcoord_backups[curr_terminal],ycoord_backups[curr_terminal]);
 ```
 
### And coding style tests

Explain what these tests test and why

```
Give an example
```

## Deployment

Add additional notes about how to deploy this on a live system

## Built With

* [Dropwizard](http://www.dropwizard.io/1.0.2/docs/) - The web framework used
* [Maven](https://maven.apache.org/) - Dependency Management
* [ROME](https://rometools.github.io/rome/) - Used to generate RSS Feeds

## Contributing

Please read [CONTRIBUTING.md](https://gist.github.com/PurpleBooth/b24679402957c63ec426) for details on our code of conduct, and the process for submitting pull requests to us.

## Versioning

We use [SemVer](http://semver.org/) for versioning. For the versions available, see the [tags on this repository](https://github.com/your/project/tags). 

## Authors

* **Billie Thompson** - *Initial work* - [PurpleBooth](https://github.com/PurpleBooth)

See also the list of [contributors](https://github.com/your/project/contributors) who participated in this project.

## License

This project is licensed under the MIT License - see the [LICENSE.md](LICENSE.md) file for details

## Acknowledgments

* Hat tip to anyone who's code was used
* Inspiration
* etc
