---
layout: post
title:  "SystemVerilog constraint of Random Input Array Distribution to Queues"
---

This is an very interesting SV constraint problem I came across in a FAANG design verification screen. Inside a SV class, given a input associative array, and three queues, after `class.randomize()` function call, would like to randomly distribute elements in  associative array to three queues. 

I struggled a lot with the this interview question. After some experimentation I think the challenge are what are the random variables in the class, what constraints are required and how to distribute and populate the output queues. 
My approach is to define random queue that define and assignment of `input_array` in the same index. I will use `post_randomize()` function to populate elements of three queues in input_array based on the assingment from `queue_assignments`

```verilog
class Packet;
    // Assuming the elements are of type int
    int input_array[];
    int queue1[$], queue2[$], queue3[$];

    // Array to keep track of which queue each element goes to
    rand int queue_assignments[];

    // Constructor to initialize the input array
    function new(int arr[]);
        input_array = arr;
        queue_assignments = new[arr.size()];
    endfunction

    // Constraint to distribute elements
    constraint distribute_elements {
        // Ensure each element in input_array is assigned to a queue
        foreach (input_array[i]) {
            queue_assignments[i] >= 1;
            queue_assignments[i] <= 3;
        }
    }

    // Post-randomize function to actually distribute elements
    function void post_randomize();
      foreach (input_array[i]) begin
            case(queue_assignments[i])
                1: queue1.push_back(input_array[i]);
                2: queue2.push_back(input_array[i]);
                3: queue3.push_back(input_array[i]);
                default: $fatal("Invalid queue assignment");
            endcase
      end
    endfunction
endclass
```

Sample testbench and simulation results:
```verilog
module tb;
  Packet pkt;
  bit[3:0] num;
  // Create a new packet, randomize it and display contents
  initial begin
    pkt = new({1, 2, 3, 4, 5, 6, 7, 8, 9});
    if(!pkt.randomize()) begin
        $error("fail to rand");
    end
    $display("queue1", pkt.queue1);
    $display("queue2", pkt.queue2);
    $display("queue3", pkt.queue3);
  end 
endmodule
```

```
queue1'{1, 3, 5, 6, 8} 
queue2'{9} 
queue3'{2, 4, 7} 
           V C S   S i m u l a t i o n   R e p o r t 
Time: 0 ns
CPU Time:      0.420 seconds;       Data structure size:   0.0Mb
```