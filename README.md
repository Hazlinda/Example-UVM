# Example-UVM

Will verify the Switch RTL core using UVM in SystemVerilog.

Following are the steps we follow to verify the Switch RTL core.
Building the Verification Environment. We will build the Environment in Multiple phases, so it will be easy to lean step by step. In this verification environment, It will not use agents and monitors to make this tutorial simple and easy.

# Phase 1 : Top
We will develop the interfaces, and connect it to DUT in top module.

# Phase 2 : Configuration
We will develop the Configuration class.


      'ifndef GUARD_CONFIGURATION 
      'define GUARD_CONFIGURATION

      class Configuration extends uvm_object;

      virtual input_interface.IP input_intf;
      virtual mem_interface.MEM mem_intf;
      virtual output_interface.OP output_intf;

      bit [7:0] device_add[4];

      virtual function uvm_object create (string name="");
      Configuration t = new();
        t.device_add = this.device_add;
        t.input_intf = this.input_intf;
        t.mem_intf = this.output_intf;
      return t;
      endfunction : create

      endcase : Configuration
      'endif
      
# Phase 3 : Environment and Testcase

We will develop the Environment class and Simple testcase and simulate them.

       `ifndef GUARD_ENV
       `define GUARD_ENV
       
        class Environment extends uvm_env;

        `uvm_component_utils(Environment)

        Sequencer Seqncr;
        Driver Drvr;
        Receiver Rcvr[4];
        Scoreboard Sbd;

        function new(string name , uvm_component parent = null);
          super.new(name, parent);
        endfunction: new


        virtual function void build();
          super.build();
        uvm_report_info(get_full_name(),"START of build ",UVM_LOW);

        Drvr   = Driver::type_id::create("Drvr",this);
        Seqncr = Sequencer::type_id::create("Seqncr",this);

        foreach(Rcvr[i]) begin
            Rcvr[i]   = Receiver::type_id::create($psprintf("Rcvr0",i),this);
            Rcvr[i].id = i;
        end

        Sbd = Scoreboard::type_id::create("Sbd",this);

        uvm_report_info(get_full_name(),"END of build ",UVM_LOW);
        endfunction

        virtual function void connect();
          super.connect();
        uvm_report_info(get_full_name(),"START of connect ",UVM_LOW);

        Drvr.seq_item_port.connect(Seqncr.seq_item_export);

        Drvr.Drvr2Sb_port.connect(Sbd.Drvr2Sb_port);

        foreach(Rcvr[i])
           Rcvr[i].Rcvr2Sb_port.connect(Sbd.Rcvr2Sb_port);

        uvm_report_info(get_full_name(),"END of connect ",UVM_LOW);
        endfunction
  
        endclass : Environment
        `endif

# Phase 4 : Packet


# Phase 5 : Sequencer and Sequence

In this phase we will develop Sequence and Sequencer. 

A sequence is series of transaction and sequencer is used to for controlling the flow of transaction generation. 
A sequence of transaction (which we already developed in previous phase) is defined by extending uvm_sequence class. uvm_sequencer does the generation of this sequence of transaction, uvm_driver takes the transaction from Sequencer and processes the packet/ drives to other component or to DUT. 

Sequencer

        `ifndef GUARD_SEQUENCER
        `define GUARD_SEQUENCER

        class Sequencer extends uvm_sequencer #(Packet);

        Configuration cfg;

        `uvm_sequencer_utils(Sequencer)

        function new (string name, uvm_component parent);
            super.new(name, parent);
            `uvm_update_sequence_lib_and_item(Packet)
        endfunction : new

        virtual function void end_of_elaboration();
            uvm_object tmp;
            assert(get_config_object("Configuration",tmp));
            $cast(cfg,tmp);
        endfunction

        endclass : Sequencer
        `endif

Sequence

        class Seq_device0_and_device1 extends uvm_sequence #(Packet);

          function new(string name = "Seq_device0_and_device1");
            super.new(name);
          endfunction : new

          Packet item;

          `uvm_sequence_utils(Seq_device0_and_device1, Sequencer)

          virtual task body();
            forever begin
              `uvm_do_with(item, {da == p_sequencer.cfg.device_add[0];} );
              `uvm_do_with(item, {da == p_sequencer.cfg.device_add[1];} );
            end
          endtask : body

          endclass : Seq_device0_and_device1

          class Seq_constant_length extends uvm_sequence #(Packet);

          function new(string name = "Seq_constant_length");
            super.new(name);
          endfunction : new

          Packet item;

          `uvm_sequence_utils(Seq_constant_length, Sequencer)

          virtual task body();
              forever begin
              `uvm_do_with(item, {length == 10; da == p_sequencer.cfg.device_add[0];} );
          end
          endtask : body

        endclass : Seq_constant_length
        
# Phase 6 : Driver

In this phase we will develop the driver. Driver is defined by extending uvm_driver. Driver takes the transaction from the sequencer using seq_item_port. This transaction will be driven to DUT as per the interface specification. After driving the transaction to DUT, it sends the transaction to scoreboard using uvm_analysis_port. 

In driver class, we will also define task for resetting DUT and configuring the DUT. After completing the driver class implementation, we will instantiate it in environment class and connect the sequencer to it. We will also update the test case and run the simulation to check the implementation which we did till now. 

      'ifndef GUARD_DRIVER 
      'define GUARD_DRIVER
      
      class Driver extends uvm_driver #(Packet);
          Configuration cfg;
          
          virtual input_interface.IP input_intf; 
          virtual mem_interface.MEM mem_intf;
      
          uvm_analysis_port #(Packet) Drvr2Sb_port;
          
          'uvm_component_utils(Driver)
          
          function new (string name = "", uvm_component parent = null); 
            super.new(name , parent); 
          endfunction : new
      
          virtual function void build(); 
            super.build(); 
            Drvr2SB_port = new ("Drvr2Sb", this); 
          endfunction : build
      
          virtual function void end_of_elaboration();
            uvm_object tmp;
            super.end_of_elaboration();
            assert (get_config_object("Configuration",tmp));
            $cast (cfg,tmp);
            this.input_intf = cfg.input_intf;
            this.mem_intf = cfg.mem_intf;
          endfunction : end_of_elaboration
  
          virtual task run();
            Packet pkt;
            @(input_intf.cb);
            reset_dut();
            cfg_dut();
            forever begin
              seq_item_port.get_next_item(pkt);
              Drvr2Sb_port.write(pkt);
              @(input_intf.cb);
              drive(pkt);
              @(input_intf.cb);
              seq_item_port.item_done();
            end
          endtask:run
  
          virtual task reset_dut();
            uvm_report_info(get_full_name(), "Start of reset_dut() method", UVM_LOW);
            mem_intf.cb.mem_data 		<=0;
            mem_intf.cb.mem_add  		<=0;
            mem_intf.cb.mem_en   		<=0;
            mem_intf.cb.mem_rd_wr  		<=0;
            input_intf.cb.data_in		<=0;
            input_intf.cb.data_status 	<=0;
    
            input_intf.reset			<=1;
            repeat (4) @input_intf.clock;
            input_intf.reset			<=0;
    
            uvm_report_info(get_full_name(), "End of reset_dut() method",UVM_LOW);
          endtask : reset_dut
  
          virtual task cfg_dut();
            uvm_report_info(get_full_name(), "Start of cfg_dut() method",UVM_LOW);
            mem_intf.cb.mem_en			<=1;
            @(mem_intf.cb);
            mem_intf.cb.mem_rd_wr		<=1;
    
          foreach (cfg.device_add[i]) begin
      
            @(mem_intf.cb);
            mem_intf.cb.mem_add	<=i;
            mem_intf.cb.mem_data	<=cfg.device_add[i];
            uvm_report_info(get_full_name(),$psprintf("Port %0d Address %h",i, cfg.device_add[i]),UVM_LOW);
          end
          
            @(mem_intf.cb); 
            mem_intf.cb.mem_en <=0; 
            mem_intf.cb.mem_rd_wr <= 0; 
            mem_intf.cb.mem_add <= 0; 
            mem_intf.cb.mem_data <= 0; 

            uvm_report_info(get_full_name(),"End of cfg_dut() method ",UVM_LOW); 
          endtask : cfg_dut 

          virtual task drive(Packet pkt); 
            byte unsigned bytes[]; 
            int pkt_len; 
            pkt_len = pkt.pack_bytes(bytes); 
            uvm_report_info(get_full_name(),"Driving packet ...",UVM_LOW); 

            foreach(bytes[i]) 
              begin 
              @(input_intf.cb); 
              input_intf.data_status <= 1 ; 
              input_intf.data_in <= bytes[i]; 
            end 

              @(input_intf.cb); 
              input_intf.data_status <= 0 ; 
              input_intf.data_in <= 0; 
              repeat(2) @(input_intf.cb); 
            endtask : drive 
         endclass:Driver
         
         'endif
   
   # Phase 7 : Receiver
   
      `ifndef GUARD_RECEIVER
      `define GUARD_RECEIVER

      class Receiver extends uvm_component;

         virtual output_interface.OP output_intf;

         Configuration cfg;

         integer id;

         uvm_analysis_port #(Packet) Rcvr2Sb_port;

         `uvm_component_utils(Receiver)

         function new (string name, uvm_component parent);
            super.new(name, parent);
         endfunction : new

        virtual function void build();
            super.build();
          Rcvr2Sb_port = new("Rcvr2Sb", this);
        endfunction : build

        virtual function void end_of_elaboration();
          uvm_object tmp;
            super.end_of_elaboration();
          assert(get_config_object("Configuration",tmp));
          $cast(cfg,tmp);
          output_intf = cfg.output_intf[id];
        endfunction : end_of_elaboration

        virtual task run();
        Packet pkt;
            fork
            forever
            begin
                // declare the queue and dynamic array here 
                // so they are automatically allocated for every packet
                bit [7:0] bq[$],bytes[];

                repeat(2) @(posedge output_intf.clock);
                wait(output_intf.cb.ready)
                output_intf.cb.read <= 1;

                repeat(2) @(posedge output_intf.clock);
                while (output_intf.cb.ready)
                begin
                      bq.push_back(output_intf.cb.data_out);
                      @(posedge output_intf.clock);
                end
                bytes = new[bq.size()] (bq); // Copy queue into dyn array

                output_intf.cb.read <= 0;
                @(posedge output_intf.clock);
                uvm_report_info(get_full_name(),"Received packet ...",UVM_LOW);
                pkt = new();
                void'(pkt.unpack_bytes(bytes));
                Rcvr2Sb_port.write(pkt);
            end
            join
            endtask : run
            
      endclass :  Receiver
      `endif
      
   # Phase 8 : Scoreboard
         
          `ifndef GUARD_SCOREBOARD
          `define GUARD_SCOREBOARD

          `uvm_analysis_imp_decl(_rcvd_pkt)
          `uvm_analysis_imp_decl(_sent_pkt)

            class Scoreboard extends uvm_scoreboard;
              `uvm_component_utils(Scoreboard)

              Packet exp_que[$];

              uvm_analysis_imp_rcvd_pkt #(Packet,Scoreboard) Rcvr2Sb_port;
              uvm_analysis_imp_sent_pkt #(Packet,Scoreboard) Drvr2Sb_port;

              function new(string name, uvm_component parent);
                  super.new(name, parent);
                  Rcvr2Sb_port = new("Rcvr2Sb", this);
                  Drvr2Sb_port = new("Drvr2Sb", this);
              endfunction : new

              virtual function void write_rcvd_pkt(input Packet pkt);
              Packet exp_pkt;
                //  pkt.print();

                  if(exp_que.size())
                  begin
                  exp_pkt = exp_que.pop_front();
                //  exp_pkt.print();
                    if( pkt.compare(exp_pkt))
                      uvm_report_info(get_type_name(), $psprintf("Sent packet and received packet matched"), UVM_LOW);
                    else
                      uvm_report_error(get_type_name(), $psprintf("Sent packet and received packet mismatched"), UVM_LOW);
                  end
                  else
                    uvm_report_error(get_type_name(), $psprintf("No more packets to in the expected queue to compare"), UVM_LOW);
                  endfunction : write_rcvd_pkt

                virtual function void write_sent_pkt(input Packet pkt);
                exp_que.push_back(pkt);
                endfunction : write_sent_pkt

                virtual function void report();
                  uvm_report_info(get_type_name(),
                  $psprintf("Scoreboard Report \n", this.sprint()), UVM_LOW);
                endfunction : report

            endclass : Scoreboard
            `endif
         
