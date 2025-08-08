# Tutorial

Shay also shared this step-by-step on how to modify a SN76489 PSG to use blip_buf. The tutorial comes as folders with incremental changes, starting with the original source code, and ending with the fully converted PSG. The correct way to read the tutorial is to `diff` each two adjacent folders and read the corresponding notes in [readme.txt](readme.txt).

I've created diffs for each step and added it here, with the explanations taken from the `readme.txt`, as a convenience.

## 1 Using blip for output at samp

First just use blip to render the normal samples, without any rate change. Instead of outputting directly to the sample buffer, we use blip at the same time points, then read the samples at the end. This serves to be sure we've got the very basic things in place.

```diff
diff -wB 0/sn76489.cpp 1/sn76489.cpp
26a27,28
> #include "blip_buf.h"
> #include <assert.h>
44a47,52
> 		for ( int i = 0; i < 2; ++i )
> 		{
> 			chip->blip [i] = blip_new( SamplingRate / 4 );
> 			blip_set_rates( chip->blip [i], SamplingRate, SamplingRate );
> 		}
> 		
56a65,66
> 	chip->prev [0] = 0;
> 	chip->prev [1] = 0;
88a99,102
> 	if ( chip )
> 		for ( int i = 0; i < 2; ++i )
> 			blip_delete( chip->blip [i] );
> 	
187,188c201,202
< 		buffer[0][j] = 0;
< 		buffer[1][j] = 0;
---
> 		int l = 0;
> 		int r = 0;
197,198c211,212
< 					buffer[0][j] += chip->Channels[i]; /* left */
< 					buffer[1][j] +=	chip->Channels[i]; /* right */
---
> 					l += chip->Channels[i]; /* left */
> 					r +=	chip->Channels[i]; /* right */
202,203c216,217
< 					buffer[0][j] += (short)( chip->panning[i][0] * chip->Channels[i] ); /* left */
< 					buffer[1][j] += (short)( chip->panning[i][1] * chip->Channels[i] ); /* right */
---
> 					l += (short)( chip->panning[i][0] * chip->Channels[i] ); /* left */
> 					r += (short)( chip->panning[i][1] * chip->Channels[i] ); /* right */
209,210c223,224
< 				buffer[0][j] += ( chip->PSGStereo >> (i+4) & 0x1 ) * chip->Channels[i]; /* left */
< 				buffer[1][j] += ( chip->PSGStereo >>  i    & 0x1 ) * chip->Channels[i]; /* right */
---
> 				l += ( chip->PSGStereo >> (i+4) & 0x1 ) * chip->Channels[i]; /* left */
> 				r += ( chip->PSGStereo >>  i    & 0x1 ) * chip->Channels[i]; /* right */
212a227,230
> 		blip_add_delta( chip->blip [0], j, l - chip->prev [0] );
> 		blip_add_delta( chip->blip [1], j, r - chip->prev [1] );
> 		chip->prev [0] = l;
> 		chip->prev [1] = r;
289a308,314
> 	}
> 	
> 	for ( int i = 0; i < 2; ++i )
> 	{
> 		blip_end_frame( chip->blip [i], length );
> 		assert( blip_samples_avail( chip->blip [i] ) == length );
> 		blip_read_samples( chip->blip [i], buffer [i], length, 0 );
diff -wB 0/sn76489.h 1/sn76489.h
81a82,84
>     struct blip_t* blip [2]; /* delta resamplers for left and right channels */
> 	int prev [2];
> 	
```

## 2 Break out delta handling

Move delta-adding code into common function, in preparation for future changes.

```diff
diff -wB 1/sn76489.cpp 2/sn76489.cpp
65,66d64
< 	chip->prev [0] = 0;
< 	chip->prev [1] = 0;
85a84,86
> 		
> 		chip->chan_amp [i] [0] = 0;
> 		chip->chan_amp [i] [1] = 0;
172a174,212
> /* Updates channel amplitude in delta buffer. Call whenever amplitude might have changed. */
> static void UpdateChanAmplitude(SN76489_Context* chip, int i, int time)
> {
> 	/* Build stereo result */
> 	int amp [2];
> 	
> 	if ( ( ( chip->PSGStereo >> i ) & 0x11 ) == 0x11 )
> 	{
> 		// no GG stereo for this channel
> 		if ( chip->panning[i][0] == 1.0f )
> 		{
> 			amp [0] = chip->Channels[i]; /* left */
> 			amp [1] = chip->Channels[i]; /* right */
> 		}
> 		else
> 		{
> 			amp [0] = (short)( chip->panning[i][0] * chip->Channels[i] ); /* left */
> 			amp [1] = (short)( chip->panning[i][1] * chip->Channels[i] ); /* right */
> 		}
> 	}
> 	else
> 	{
> 		// GG stereo overrides panning
> 		amp [0] = ( chip->PSGStereo >> (i+4) & 0x1 ) * chip->Channels[i]; /* left */
> 		amp [1] = ( chip->PSGStereo >>  i    & 0x1 ) * chip->Channels[i]; /* right */
> 	}
> 	
> 	/* Update amplitudes in left and right buffers */
> 	for ( int j = 0; j < 2; ++j )
> 	{
> 		int delta = amp [j] - chip->chan_amp [i] [j];
> 		if ( delta != 0 )
> 		{
> 			chip->chan_amp [i] [j] = amp [j];
> 			blip_add_delta( chip->blip [j], time, delta );
> 		}
> 	}
> }
> 
201,202d240
< 		int l = 0;
< 		int r = 0;
205,230c243
< 		{
< 			if ( ( ( chip->PSGStereo >> i ) & 0x11 ) == 0x11 )
< 			{
< 				// no GG stereo for this channel
< 				if ( chip->panning[i][0] == 1.0f )
< 				{
< 					l += chip->Channels[i]; /* left */
< 					r +=	chip->Channels[i]; /* right */
< 				}
< 				else
< 				{
< 					l += (short)( chip->panning[i][0] * chip->Channels[i] ); /* left */
< 					r += (short)( chip->panning[i][1] * chip->Channels[i] ); /* right */
< 				}
< 		  }
< 			else
< 			{
< 				// GG stereo overrides panning
< 				l += ( chip->PSGStereo >> (i+4) & 0x1 ) * chip->Channels[i]; /* left */
< 				r += ( chip->PSGStereo >>  i    & 0x1 ) * chip->Channels[i]; /* right */
< 			}
< 		}
< 		blip_add_delta( chip->blip [0], j, l - chip->prev [0] );
< 		blip_add_delta( chip->blip [1], j, r - chip->prev [1] );
< 		chip->prev [0] = l;
< 		chip->prev [1] = r;
---
> 			UpdateChanAmplitude( chip, i, j );
diff -wB 1/sn76489.h 2/sn76489.h
83c83
< 	int prev [2];
---
> 	int chan_amp [4] [2];     /* current channel amplitudes in delta buffers */
```

## 3 Clock rate above sample rate

Now that delta adding is contained, raise the clock rate above the sample rate again.

```diff
diff -wB 2/sn76489.cpp 3/sn76489.cpp
50c50
< 			blip_set_rates( chip->blip [i], SamplingRate, SamplingRate );
---
> 			blip_set_rates( chip->blip [i], PSGClockValue, SamplingRate * 16 );
53c53
< 		chip->dClock=(float)PSGClockValue/16/SamplingRate;
---
> 		chip->dClock=1;
217c217,219
< 	for( j = 0; j < length; j++ )
---
> 	int clock_length = blip_clocks_needed( chip->blip [0], length );
> 	
> 	for( j = 0; j < clock_length; j++ )
245c247
< 		/* Increment clock by 1 sample length */
---
> 		/* Increment clock by 1 clock length */
325c327
< 		blip_end_frame( chip->blip [i], length );
---
> 		blip_end_frame( chip->blip [i], clock_length );
```

## 4 Eliminate old fractional sample handling

Eliminate unnecessary fractional sample handling from old clock rate code.

```diff
diff -wB 3/sn76489.cpp 4/sn76489.cpp
53d52
< 		chip->dClock=1;
93,95d91
< 
< 	/* Zero clock */
< 	chip->Clock = 0;
247,250c243,244
< 		/* Increment clock by 1 clock length */
< 		chip->Clock += chip->dClock;
< 		chip->NumClocksForSample = (int)chip->Clock;  /* truncate */
< 		chip->Clock -= chip->NumClocksForSample;      /* remove integer part */
---
> 		/* Increment clock by 1 sample length */
> 		float const NumClocksForSample = 1;
254c248
< 			chip->ToneFreqVals[i] -= chip->NumClocksForSample;
---
> 			chip->ToneFreqVals[i] -= NumClocksForSample;
260c254
< 			chip->ToneFreqVals[3] -= chip->NumClocksForSample;
---
> 			chip->ToneFreqVals[3] -= NumClocksForSample;
268c262
< 					chip->IntermediatePos[i] = ( chip->NumClocksForSample - chip->Clock + 2 * chip->ToneFreqVals[i] ) * chip->ToneFreqPos[i] / ( chip->NumClocksForSample + chip->Clock );
---
> 					chip->IntermediatePos[i] = ( NumClocksForSample + 2 * chip->ToneFreqVals[i] ) * chip->ToneFreqPos[i] / ( NumClocksForSample );
276c270
< 				chip->ToneFreqVals[i] += chip->Registers[i*2] * ( chip->NumClocksForSample / chip->Registers[i*2] + 1 );
---
> 				chip->ToneFreqVals[i] += chip->Registers[i*2] * ( NumClocksForSample / chip->Registers[i*2] + 1 );
290c284
< 				chip->ToneFreqVals[3] += chip->NoiseFreq * ( chip->NumClocksForSample / chip->NoiseFreq + 1 );
---
> 				chip->ToneFreqVals[3] += chip->NoiseFreq * ( NumClocksForSample / chip->NoiseFreq + 1 );
diff -wB 3/sn76489.h 4/sn76489.h
61,62d60
<     float Clock;
<     float dClock;
```

## 5 Break out square/noise channels

This will make it easier to optimize them.

```diff
diff -wB 4/sn76489.cpp 5/sn76489.cpp
209c209,210
< void SN76489_Update(SN76489_Context* chip, INT16 **buffer, int length)
---
> /* Updates tone amplitude in delta buffer. Call whenever amplitude might have changed. */
> static void UpdateToneAmplitude(SN76489_Context* chip, int i, int time)
211,218d211
< 	int i, j;
< 	
< 	int clock_length = blip_clocks_needed( chip->blip [0], length );
< 	
< 	for( j = 0; j < clock_length; j++ )
< 	{
< 		/* Tone channels */
< 		for ( i = 0; i <= 2; ++i )
232c225,230
< 		/* Noise channel */
---
> 	UpdateChanAmplitude(chip, i, time);
> }
> 
> /* Updates noise amplitude in delta buffer. Call whenever amplitude might have changed. */
> static void UpdateNoiseAmplitude(SN76489_Context* chip, int time)
> {
236c234
< 			chip->Channels[i] = 0;
---
> 		chip->Channels[3] = 0;
237a236,246
> 	UpdateChanAmplitude(chip, 3, time);
> }
> 
> void SN76489_Update(SN76489_Context* chip, INT16 **buffer, int length)
> {
> 	int i, j;
> 	
> 	int clock_length = blip_clocks_needed( chip->blip [0], length );
> 	
> 	for( j = 0; j < clock_length; j++ )
> 	{
240,241c249,251
< 		for ( i = 0; i <= 3; ++i )
< 			UpdateChanAmplitude( chip, i, j );
---
> 		for ( i = 0; i < 3; ++i )
> 			UpdateToneAmplitude(chip, i, j);
> 		UpdateNoiseAmplitude(chip, j);
```

## 6 Eliminate more fractional sample handling

```diff
diff -wB 5/sn76489.cpp 6/sn76489.cpp
78,80d77
< 		/* Set intermediate positions to do-not-use value */
< 		chip->IntermediatePos[i] = FLT_MIN;
< 
213,218d209
< 	{
< 		if ( chip->IntermediatePos[i] != FLT_MIN )
< 			/* Intermediate position (antialiasing) */
< 			chip->Channels[i] = (short)( PSGVolumeValues[chip->Registers[2 * i + 1]] * chip->IntermediatePos[i] );
< 		else
< 			/* Flat (no antialiasing needed) */
220d210
< 	}
253,255d242
< 		/* Increment clock by 1 sample length */
< 		float const NumClocksForSample = 1;
< 	
258c245
< 			chip->ToneFreqVals[i] -= NumClocksForSample;
---
> 			chip->ToneFreqVals[i]--;
264c251
< 			chip->ToneFreqVals[3] -= NumClocksForSample;
---
> 			chip->ToneFreqVals[3]--;
269,272c256
< 				if (chip->Registers[i*2]>PSG_CUTOFF) {
< 					/* For tone-generating values, calculate how much of the sample is + and how much is - */
< 					/* This is optimised into an even more confusing state than it was in the first place... */
< 					chip->IntermediatePos[i] = ( NumClocksForSample + 2 * chip->ToneFreqVals[i] ) * chip->ToneFreqPos[i] / ( NumClocksForSample );
---
> 				if (chip->Registers[i*2]>PSG_CUTOFF)
275c259
< 				} else {
---
> 				else
278c262
< 					chip->IntermediatePos[i] = FLT_MIN;
---
> 				chip->ToneFreqVals[i] += chip->Registers[i*2] + 1;
280,284d263
< 				chip->ToneFreqVals[i] += chip->Registers[i*2] * ( NumClocksForSample / chip->Registers[i*2] + 1 );
< 			}
< 			else
< 				/* signal no antialiasing needed */
< 				chip->IntermediatePos[i] = FLT_MIN;
294c273
< 				chip->ToneFreqVals[3] += chip->NoiseFreq * ( NumClocksForSample / chip->NoiseFreq + 1 );
---
> 				chip->ToneFreqVals[3] += chip->NoiseFreq + 1;
diff -wB 5/sn76489.h 6/sn76489.h
76d75
<     float IntermediatePos[4];   /* intermediate values used at boundaries between + and - (does not need double accuracy)*/
```

# 7 Minor tweaks before channel optimization

```diff
diff -wB 6/sn76489.cpp 7/sn76489.cpp
235,238c235
< 	for( j = 0; j < clock_length; j++ )
< 	{
< 		/* Build stereo result into buffer */
< 		/* For all 4 channels */
---
> 	/* Update in case a register changed etc. */
240,241c237,238
< 			UpdateToneAmplitude(chip, i, j);
< 		UpdateNoiseAmplitude(chip, j);
---
> 		UpdateToneAmplitude(chip, i, 0);
> 	UpdateNoiseAmplitude(chip, 0);
243,248c240,244
< 		/* Decrement tone channel counters */
< 		for ( i = 0; i <= 2; ++i )
< 			chip->ToneFreqVals[i]--;
< 	 
< 		/* Noise channel: match to tone2 or decrement its counter */
< 		if ( chip->NoiseFreq == 0x80 )
---
> 	/* Noise channel: match to tone2 if in slave mode */
> 	int NoiseFreq = chip->NoiseFreq;
> 	if ( NoiseFreq == 0x80 )
> 	{
> 		NoiseFreq = chip->Registers[2*2];
250,251c246
< 		else
< 			chip->ToneFreqVals[3]--;
---
> 	}
252a248,249
> 	for( j = 0; j < clock_length; j++ )
> 	{
254a252
> 			chip->ToneFreqVals[i]--;
261a260
> 				UpdateToneAmplitude(chip, i, j);
266a266
> 		chip->ToneFreqVals[3]--;
273c273
< 				chip->ToneFreqVals[3] += chip->NoiseFreq + 1;
---
> 				chip->ToneFreqVals[3] += NoiseFreq + 1;
303a304
> 				UpdateNoiseAmplitude(chip, j);
```

# 8 Optimize channel loops

Turn things inside-out so that loop runs once per event rather than per clock, advancing the number of clocks until the next event. Add interface to run by clock precision rather than sample as before.

```diff
diff -wB 7/sn76489.cpp 8/sn76489.cpp
229c229,230
< void SN76489_Update(SN76489_Context* chip, INT16 **buffer, int length)
---
> /* Runs noise channel for clock_length clocks */
> static void RunNoise(SN76489_Context* chip, int clock_length)
231,234d231
< 	int i, j;
< 	
< 	int clock_length = blip_clocks_needed( chip->blip [0], length );
< 	
236,237d232
< 	for ( i = 0; i < 3; ++i )
< 		UpdateToneAmplitude(chip, i, 0);
248,263c243,244
< 	for( j = 0; j < clock_length; j++ )
< 	{
< 		/* Tone channels: */
< 		for ( i = 0; i <= 2; ++i ) {
< 			chip->ToneFreqVals[i]--;
< 			if ( chip->ToneFreqVals[i] <= 0 ) {   /* If the counter gets below 0... */
< 				if (chip->Registers[i*2]>PSG_CUTOFF)
< 					/* Flip the flip-flop */
< 					chip->ToneFreqPos[i] = -chip->ToneFreqPos[i];
< 				else
< 					/* stuck value */
< 					chip->ToneFreqPos[i] = 1;
< 				UpdateToneAmplitude(chip, i, j);
< 				chip->ToneFreqVals[i] += chip->Registers[i*2] + 1;
< 			}
< 		}
---
> 	/* Time of next transition */
> 	int time = chip->ToneFreqVals[3];
265,267c246,248
< 		/* Noise channel */
< 		chip->ToneFreqVals[3]--;
< 		if ( chip->ToneFreqVals[3] <= 0 ) {
---
> 	/* Process any transitions that occur within clocks we're running */
> 	while ( time < clock_length )
> 	{
271,273d251
< 			if (chip->NoiseFreq != 0x80)
< 				/* If not matching tone2, decrement counter */
< 				chip->ToneFreqVals[3] += NoiseFreq + 1;
304c282,335
< 				UpdateNoiseAmplitude(chip, j);
---
> 			UpdateNoiseAmplitude(chip, time);
> 		}
> 		
> 		/* Advance to time of next transition */
> 		time += NoiseFreq + 1;
> 	}
> 	
> 	/* Calculate new value for register, now that next transition is past number of clocks we're running */
> 	chip->ToneFreqVals[3] = time - clock_length;
> }
> 
> /* Runs tone channel for clock_length clocks */
> static void RunTone(SN76489_Context* chip, int i, int clock_length)
> {
> 	/* Update in case a register changed etc. */
> 	UpdateToneAmplitude(chip, i, 0);
> 	
> 	/* Time of next transition */
> 	int time = chip->ToneFreqVals[i];
> 	
> 	/* Process any transitions that occur within clocks we're running */
> 	while ( time < clock_length )
> 	{
> 		if (chip->Registers[i*2]>PSG_CUTOFF)
> 			/* Flip the flip-flop */
> 			chip->ToneFreqPos[i] = -chip->ToneFreqPos[i];
> 		else
> 			/* stuck value */
> 			chip->ToneFreqPos[i] = 1;
> 		
> 		UpdateToneAmplitude(chip, i, time);
> 		
> 		/* Advance to time of next transition */
> 		time += chip->Registers[i*2] + 1;
> 	}
> 	
> 	/* Calculate new value for register, now that next transition is past number of clocks we're running */
> 	chip->ToneFreqVals[i] = time - clock_length;
> }
> 
> /* Runs sound for N clocks */
> void SN76489_RunClocks(SN76489_Context* chip, int Clocks)
> {
> 	if ( Clocks > 0 )
> 	{
> 		/* Run noise first, since it might use current value of third tone frequency counter */
> 		RunNoise(chip, Clocks);
> 		
> 		/* Run tone channels */
> 		for ( int i = 0; i <= 2; ++i )
> 			RunTone(chip, i, Clocks);
> 		
> 		for ( int i = 0; i < 2; ++i )
> 			blip_end_frame( chip->blip [i], Clocks );
306a338,345
> 
> /* Runs sound for N samples */
> void SN76489_RunSamples(SN76489_Context* chip, int Samples)
> {
> 	/* Find out how many clocks are needed to make this many more samples available */
> 	int Clocks = blip_clocks_needed( chip->blip [0], Samples + SN76489_SamplesAvail( chip ) );
> 	
> 	SN76489_RunClocks( chip, Clocks );
308a348,360
> /* Returns number of samples available for reading */
> int SN76489_SamplesAvail(SN76489_Context* chip)
> {
> 	return blip_samples_avail( chip->blip [0] );
> }
> 
> void SN76489_Update(SN76489_Context* chip, INT16 **buffer, int length)
> {
> 	/* Find out how many clocks are needed to make this many samples available */
> 	int Clocks = blip_clocks_needed( chip->blip [0], length );
> 	SN76489_RunClocks( chip, Clocks );
> 	
> 	/* Read samples into output buffer */
311,312c363
< 		blip_end_frame( chip->blip [i], clock_length );
< 		assert( blip_samples_avail( chip->blip [i] ) == length );
---
> 		assert( blip_samples_avail( chip->blip [i] ) >= length );
diff -wB 7/sn76489.h 8/sn76489.h
108a109,117
> /* Runs sound for N clocks. Any samples generated are buffered internally. */
> void SN76489_RunClocks(SN76489_Context*, int Clocks);
> 
> /* Runs sound for N samples. Any samples generated are buffered internally. */
> void SN76489_RunSamples(SN76489_Context*, int Samples);
> 
> /* Returns number of samples in internal buffer */
> int SN76489_SamplesAvail(SN76489_Context*);
> 
```
