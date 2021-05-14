# General description or introduction of the problem and your solution

This homework requires us to implement gaussian blur in systemC and uses stratus to transfer our code to verilog code for High-level Synthesis. Moreover, we are asked to implement to different fifo situations. One is to split the rgb to different channels, the other is not to split the channels.

# Implementation details (data structure, flows and algorithms)
## not_split
* main.cpp
```

extern void esc_elaborate()
{
	sys = new System("sys", esc_argv(1), esc_argv(2));
}
extern void esc_cleanup()
{
	delete sys;
}
int sc_main(int argc, char **argv) {
  if ((argc < 3) || (argc > 4)) {
    cout << "No arguments for the executable : " << argv[0] << endl;
    cout << "Usage : >" << argv[0] << " in_image_file_name out_image_file_name"
         << endl;
    return 0;
  }
	esc_initialize(argc, argv);
	esc_elaborate();
	sc_start();
	esc_cleanup();
	std::cout<< "Simulated time == " << sc_core::sc_time_stamp() << std::endl;
  return 0;
}

```
This main.cpp new an class called System to start doing gaussian filter, and delete the system after the program is done.

* system.cpp
```

System::System( sc_module_name n, string input_bmp, string output_bmp ): sc_module( n ), 
	tb("tb"), sobel_filter("sobel_filter"), clk("clk", CLOCK_PERIOD, SC_NS), rst("rst"), _output_bmp(output_bmp)
{
	tb.i_clk(clk);
	tb.o_rst(rst);
	sobel_filter.i_clk(clk);
	sobel_filter.i_rst(rst);
	tb.o_rgb(rgb);
	tb.i_result(result);
	sobel_filter.i_rgb(rgb);
	sobel_filter.o_result(result);

  tb.read_bmp(input_bmp);
}

System::~System() {
  tb.write_bmp(_output_bmp);
}


```

System.cpp new testbench and gaussian_filter, binding both togather using cynw_p2p.

* testbench.cpp
```
Testbench::Testbench(sc_module_name n) : sc_module(n), output_rgb_raw_data_offset(54)
{
  SC_THREAD(feed_rgb);
  sensitive << i_clk.pos();
  dont_initialize();
  SC_THREAD(fetch_result);
  sensitive << i_clk.pos();
  dont_initialize();
}

...

void Testbench::feed_rgb()
{
  unsigned int x, y, i, v, u; // for loop counter
  unsigned char R, G, B;      // color of R, G, B
  int adjustX, adjustY, xBound, yBound;
  n_txn = 0;
  max_txn_time = SC_ZERO_TIME;
  min_txn_time = SC_ZERO_TIME;
  total_txn_time = SC_ZERO_TIME;

#ifndef NATIVE_SYSTEMC
  o_rgb.reset();
#endif
  o_rst.write(false);
  wait(5);
  o_rst.write(true);
  wait(1);
  total_start_time = sc_time_stamp();
  for (y = 0; y != height; ++y)
  {
    for (x = 0; x != width; ++x)
    {
      adjustX = (MASK_X % 2) ? 1 : 0; // 1
      adjustY = (MASK_Y % 2) ? 1 : 0; // 1
      xBound = MASK_X / 2;            // 1
      yBound = MASK_Y / 2;            // 1

      for (v = -yBound; v != yBound + adjustY; ++v)
      { //-1, 0, 1
        for (u = -xBound; u != xBound + adjustX; ++u)
        { //-1, 0, 1
          if (x + u >= 0 && x + u < width && y + v >= 0 && y + v < height)
          {
            R = *(source_bitmap +
                  bytes_per_pixel * (width * (y + v) + (x + u)) + 2);
            G = *(source_bitmap +
                  bytes_per_pixel * (width * (y + v) + (x + u)) + 1);
            B = *(source_bitmap +
                  bytes_per_pixel * (width * (y + v) + (x + u)) + 0);
          }
          else
          {
            R = 0;
            G = 0;
            B = 0;
          }
          sc_dt::sc_uint<24> rgb;
          rgb.range(7, 0) = R;
          rgb.range(15, 8) = G;
          rgb.range(23, 16) = B;
#ifndef NATIVE_SYSTEMC
          o_rgb.put(rgb);
#else
          o_rgb.write(rgb);
#endif
        }
      }
    }
  }
}


void Testbench::fetch_result()
{
  unsigned int x, y;     // for loop counter
  unsigned int R, G, B; // color of R, G, B
  sc_dt::sc_uint<36> i_rgb;
#ifndef NATIVE_SYSTEMC
  i_result.reset();
#endif
  wait(5);
  wait(1);
  for (y = 0; y != height; ++y)
  {
    for (x = 0; x != width; ++x)
    {
#ifndef NATIVE_SYSTEMC
      i_rgb = i_result.get();
#else
      total = i_result.read();
#endif
      R = i_rgb.range(11, 0);
      G = i_rgb.range(23, 12);
      B = i_rgb.range(35, 24);
      //std::cout << "Orgin R = " << (unsigned int) R << std::endl;
      //std::cout << "R = " << ((unsigned int) R ) / 16 << std::endl;
      *(target_bitmap + bytes_per_pixel * (width * y + x) + 2) = R / 16;
      *(target_bitmap + bytes_per_pixel * (width * y + x) + 1) = G / 16;
      *(target_bitmap + bytes_per_pixel * (width * y + x) + 0) = B / 16;
    }
  }
  total_run_time = sc_time_stamp() - total_start_time;
  sc_stop();
}

```

Testbench.cpp has two functions that are different with previous hw. feed_rgb responsible to sent rgb data to filter. fetch_rgb responsible to get result from filter.

* sobelfilter.cpp

```

void SobelFilter::do_filter()
{
	{
#ifndef NATIVE_SYSTEMC
		HLS_DEFINE_PROTOCOL("main_reset");
		i_rgb.reset();
		o_result.reset();
#endif
		wait();
	}
	while (true)
	{
		for (unsigned int i = 0; i < MASK_N; ++i)
		{
			HLS_CONSTRAIN_LATENCY(0, 1, "lat00");
			val[i] = 0;
		}
		unsigned int red = 0.0, green = 0.0, blue = 0.0;
		for (unsigned int v = 0; v < MASK_Y; ++v)
		{
			for (unsigned int u = 0; u < MASK_X; ++u)
			{
				sc_dt::sc_uint<24> rgb;
#ifndef NATIVE_SYSTEMC
				{
					HLS_DEFINE_PROTOCOL("input");
					rgb = i_rgb.get();
					wait();
				}
#else
				rgb = i_rgb.read();
#endif
				unsigned char _red = rgb.range(7, 0);
				unsigned char _green = rgb.range(15, 8);
				unsigned char _blue = rgb.range(23, 16);
				//unsigned char grey = (rgb.range(7, 0) + rgb.range(15, 8) + rgb.range(23, 16)) / 3;

				red += _red * mask[u][v];
				green += _green * mask[u][v];
				blue += _blue * mask[u][v];
			}
		}
		//std::cout << "o_red = " << o_red << " red = " << (unsigned char) ((int) (red * 1/16)) << std::endl;
		sc_dt::sc_uint<36> o_rgb;
		o_rgb.range(11, 0) = (red);
		o_rgb.range(23, 12) = (green);
		o_rgb.range(35, 24) =  (blue);
#ifndef NATIVE_SYSTEMC
		{
			HLS_DEFINE_PROTOCOL("output");
			o_result.put(o_rgb);
			wait();
		}
#else
		o_result.write(total);
#endif
	}
}


```

Filter has very similar code with previous hw with some adjustment about data transfering.
## split

* system.cpp
```
tb.o_r(r);
tb.o_g(g);
tb.o_b(b);
tb.i_result(result);
//sobel_filter.i_rgb(rgb);
sobel_filter.i_r(r);
sobel_filter.i_g(g);
sobel_filter.i_b(b);

```

The code are just like previous one with adjusting the data connection way to different channels.

# Result
## not_split
* BASIC:
    *  stimulation time : 30801910 ns
    *  total area : 1830
        * resource metrics : 508
        * mux metrics : 449
        * register metrics : 811
        * memory metrics : 0
        * timing metrics : 7.2 (worst slack) 
* DPA:
    *  stimulation time : 24903670 ns
    *  total area : 1628
        * resource metrics : 462
        * mux metrics : 349
        * register metrics : 769
        * memory metrics : 0
        * timing metrics : 8.0 (worst slack) 
## split
* BASIC:
    *  stimulation time : 42598390 ns
    *  total area : 1908
        * resource metrics : 516
        * mux metrics : 366
        * register metrics : 956
        * memory metrics : 0
        * timing metrics : 7.2 (worst slack) 
* DPA:
    *  stimulation time : 36700150 ns
    *  total area : 1744
        * resource metrics : 470
        * mux metrics : 289
        * register metrics : 930
        * memory metrics : 0
        * timing metrics : 8.0 (worst slack) 

# Discussion
This homework is quite hard, but it is without a doubet a good practice for us to learn systemC. Besides, we have to write our report in English and submit our homework to GitHub, which are not familiar to me. Anyway, thank you for providing us this challenge.
