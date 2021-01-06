# General
## Experiments
### How to get real physic memory size of an iOS progress.
```objective-c
    static BOOL isAlloc = NO;
    static int *bytes = NULL; // Step one hold 512 MB in memory
     if (!isAlloc) {
         isAlloc = YES;
         bytes = (int*)malloc(512 * 1024 * 1024);
         int size = 512 * 1024 * 1024 / 4;
         for (int i = 0 ; i < size; i++) {
             bytes[i] = 3;
         }
     }
    
    // These are codes used to retrieve memory size info.
    task_vm_info_data_t taskInfo;
    mach_msg_type_number_t infoCount = TASK_VM_INFO_COUNT;
    kern_return_t kernReturn = task_info(mach_task_self(),
                                         TASK_VM_INFO,
                                         (task_info_t)&taskInfo,
                                         &infoCount);
    
    NSByteCountFormatter *formatter = [NSByteCountFormatter new];
    NSString *label = [NSString stringWithFormat:@"res: %@, phys: %@"
                       , [formatter stringFromByteCount:taskInfo.resident_size]
                       , [formatter stringFromByteCount:taskInfo.phys_footprint]];
    NSLog(@"%@", label); // THe print out is res: 624.5 MB, phys: 545.9 MB while XCode is showing Memory occupied 521.1MB
```

Here are an explanation of **__resident_size__** and **__phys_footprint__** in the [source code](https://github.com/apple/darwin-xnu?spm=ata.13261165.0.0.2ba95e2cCrsS5m): (osfmk/kern/task.c)
```c
 /*
 * Task ledgers
 * ------------
 *
 * phys_footprint
 *   Physical footprint: This is the sum of:
 *     + (internal - alternate_accounting)
 *     + (internal_compressed - alternate_accounting_compressed)
 *     + iokit_mapped
 *     + purgeable_nonvolatile
 *     + purgeable_nonvolatile_compressed
 *     + page_table
 *
 * internal
 *   The task's anonymous memory, which on iOS is always resident.
 *
 * internal_compressed
 *   Amount of this task's internal memory which is held by the compressor.
 *   Such memory is no longer actually resident for the task [i.e., resident in its pmap],
 *   and could be either decompressed back into memory, or paged out to storage, depending
 *   on our implementation.
 *
 * iokit_mapped
 *   IOKit mappings: The total size of all IOKit mappings in this task, regardless of
     clean/dirty or internal/external state].
 *
 * alternate_accounting
 *   The number of internal dirty pages which are part of IOKit mappings. By definition, these pages
 *   are counted in both internal *and* iokit_mapped, so we must subtract them from the total to avoid
 *   double counting.
 */
 ```

 Inside, **internal** is the same as **resident_size**, and **phys_footprint** is more closer to the real memory size.. Also, please be aware that [Jetsam](https://developer.apple.com/documentation/xcode/diagnosing_issues_using_crash_reports_and_device_logs/identifying_high-memory_use_with_jetsam_event_reports) is using **phys_footprint**.

 We could use another experience to tell the differences of **__resident_size__** and **__phys_footprint__**.
 ```objective-c
    self.currentTimer = [NSTimer scheduledTimerWithTimeInterval:2.0 repeats:YES block:^(NSTimer * _Nonnull timer) {
        static BOOL isAlloc = NO;
        static int *bytes = NULL; // We occupied 100 MB for every 2 seconds
        if (!isAlloc) {
            bytes = (int*)malloc(100 * 1024 * 1024);
            int size = 100 * 1024 * 1024 / 4;
            for (int i = 0 ; i < size; i++) {
                bytes[i] = 3;
            }
        }
        
        task_vm_info_data_t taskInfo;
        mach_msg_type_number_t infoCount = TASK_VM_INFO_COUNT;
        kern_return_t kernReturn = task_info(mach_task_self(),
                                             TASK_VM_INFO,
                                             (task_info_t)&taskInfo,
                                             &infoCount);
        
        NSByteCountFormatter *formatter = [NSByteCountFormatter new];
        NSString *label = [NSString stringWithFormat:@"res: %@, phys: %@"
                           , [formatter stringFromByteCount:taskInfo.resident_size]
                           , [formatter stringFromByteCount:taskInfo.phys_footprint]];
        NSLog(@"%@", label);
    }];

    // And the Print out is:
    // 2021-01-06 21:03:11.875448+0800 res: 164.4 MB, phys: 118.3 MB
    // 2021-01-06 21:03:13.887627+0800 res: 269.1 MB, phys: 223.3 MB
    // 2021-01-06 21:03:15.865496+0800 res: 269.9 MB, phys: 328.1 MB
    // 2021-01-06 21:03:17.863876+0800 res: 290 MB, phys: 433.2 MB
    // 2021-01-06 21:03:19.888331+0800 res: 356.3 MB, phys: 538 MB
    // 2021-01-06 21:03:21.885885+0800 res: 372.3 MB, phys: 642.9 MB
    // 2021-01-06 21:03:23.884214+0800 res: 405.3 MB, phys: 747.9 MB
    // 2021-01-06 21:03:25.858744+0800 res: 418.6 MB, phys: 852.8 MB
    // 2021-01-06 21:03:27.889009+0800 res: 420.9 MB, phys: 957.7 MB
    // 2021-01-06 21:03:29.888773+0800 res: 433 MB, phys: 1.06 GB
    // 2021-01-06 21:03:31.887442+0800 res: 433.9 MB, phys: 1.17 GB
    // 2021-01-06 21:03:33.902337+0800 res: 462.9 MB, phys: 1.27 GB
    // 2021-01-06 21:03:35.887407+0800 res: 505.5 MB, phys: 1.38 GB
    // 2021-01-06 21:03:37.903741+0800 res: 521.7 MB, phys: 1.48 GB
 ```