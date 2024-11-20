````rust
/* Written by Nick Ilhan Atamgüc <nickatamguec@outlook.com> */

// This is a theory to illustrate the steps of "neon_movemask_epu8" for "little endian" and "big endian".
// How "big endian" works in practice cannot be tested, but if I have understood everything correctly, the example shown below should be correct.

// ================================================================= SOURCE CODE =================================================================
use core::arch::aarch64 as arm;

// bytes = [0xFF, 0xFF, 0xFF, 0xFF, 0xFF, 0x0, 0xFF, 0xFF, 0xFF, 0xFF, 0x0, 0x0, 0xFF, 0xFF, 0x0, 0x0]
// value = arm::vld1q_u8(bytes.as_ptr())
unsafe fn neon_movemask_epu8(value: arm::uint8x16_t) -> u16 {
    let shift_u8 = arm::vshrq_n_u8(value, 7);
    let vec_8x16 = arm::vreinterpretq_u16_u8(shift_u8);

    let shift_u16 = arm::vsraq_n_u16(vec_8x16, vec_8x16, 7);
    let vec_4x32 = arm::vreinterpretq_u32_u16(shift_u16);

    let shift_u32 = arm::vsraq_n_u32(vec_4x32, vec_4x32, 14);
    let vec_2x64 = arm::vreinterpretq_u64_u32(shift_u32);

    let shift_u64 = arm::vsraq_n_u64(vec_2x64, vec_2x64, 28);
    let vec_16x8 = arm::vreinterpretq_u8_u64(shift_u64);

    let (low, high): (u8, u8);
    #[cfg(target_endian = "little")]
    {
        (low, high) = (
            arm::vgetq_lane_u8(vec_16x8, 0),
            arm::vgetq_lane_u8(vec_16x8, 8),
        );
    }
    #[cfg(target_endian = "big")]
    {
        let reversed = arm::vrbitq_u8(vec_16x8);
        (low, high) = (
            arm::vgetq_lane_u8(reversed, 7),
            arm::vgetq_lane_u8(reversed, 15),
        );
    }

    ((high as u16) << 8) | (low as u16)
}
// 
// ================================================================ LITTLE ENDIAN ================================================================
// Note: The bytes in the SIMD vector are swapped. The element (0) will be on the right and the element (15) on the left
// These are our 16 bytes before and after we put them into a SIMD vector (vld1q_u8):
// 11111111 11111111 11111111 11111111 11111111 00000000 11111111 11111111 11111111 11111111 00000000 00000000 11111111 11111111 00000000 00000000
// <<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<< AFTER >>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>
// 00000000 00000000 11111111 11111111 00000000 00000000 11111111 11111111 11111111 11111111 00000000 11111111 11111111 11111111 11111111 11111111
// \_>>7__/ \_>>7__/ \_>>7__/ \_>>7__/ \_>>7__/ \_>>7__/ \_>>7__/ \_>>7__/ \_>>7__/ \_>>7__/ \_>>7__/ \_>>7__/ \_>>7__/ \_>>7__/ \_>>7__/ \_>>7__/
// 00000000 00000000 00000001 00000001 00000000 00000000 00000001 00000001 00000001 00000001 00000000 00000001 00000001 00000001 00000001 00000001
// \______>>7______/ \______>>7______/ \______>>7______/ \______>>7______/ \______>>7______/ \______>>7______/ \______>>7______/ \______>>7______/
//     00000000          00000011          00000000          00000011          00000011          00000001          00000011          00000011
//          __|_____          __|_____          __|_____          __|_____          __|_____          __|_____          __|_____          __|_____
// 00000000 00000000 00000001 00000011 00000000 00000000 00000001 00000011 00000001 00000011 00000000 00000001 00000001 00000011 00000001 00000011
// \______________>>14_______________/ \______________>>14_______________/ \______________>>14_______________/ \______________>>14_______________/
//              00000011______                      00000011______                      00001101______                      00001111______
//                            |_______                            |_______                            |_______                            |_______
// 00000000 00000000 00000001 00000011 00000000 00000000 00000001 00000011 00000001 00000011 00000000 00001101 00000001 00000011 00000001 00001111
// \________________________________>>28_________________________________/ \________________________________>>28_________________________________/
//                                00110011________________________                                        11011111________________________
//                                                                |_______                                                                |_______
// 00000000 00000000 00000001 00000011 00000000 00000000 00000001 00110011 00000001 00000011 00000000 00001101 00000001 00000011 00000001 11011111
// \__15__/ \__14__/ \__13__/ \__12__/ \__11__/ \__10__/ \__09__/ \__08__/ \__07__/ \__06__/ \__05__/ \__04__/ \__03__/ \__02__/ \__01__/ \__00__/
//                                                                |      |________________________________________________________________|      |
//                                                                |______________________________         |         _____________________________|
//                                                                                               |00110011|11011111|
//                                                                                               |  HIGH  |  LOW   |
// 
// ================================================================= BIG ENDIAN ==================================================================
// These are our 16 bytes after we put them into a SIMD vector:
// 11111111 11111111 11111111 11111111 11111111 00000000 11111111 11111111 11111111 11111111 00000000 00000000 11111111 11111111 00000000 00000000
// \_>>7__/ \_>>7__/ \_>>7__/ \_>>7__/ \_>>7__/ \_>>7__/ \_>>7__/ \_>>7__/ \_>>7__/ \_>>7__/ \_>>7__/ \_>>7__/ \_>>7__/ \_>>7__/ \_>>7__/ \_>>7__/
// 00000001 00000001 00000001 00000001 00000001 00000000 00000001 00000001 00000001 00000001 00000000 00000000 00000001 00000001 00000000 00000000
// \______>>7______/ \______>>7______/ \______>>7______/ \______>>7______/ \______>>7______/ \______>>7______/ \______>>7______/ \______>>7______/
//     00000011          00000011          00000010          00000011          00000011          00000000          00000011          00000000
//          __|_____          __|_____          __|_____          __|_____          __|_____          __|_____          __|_____          __|_____
// 00000001 00000011 00000001 00000011 00000001 00000010 00000001 00000011 00000001 00000011 00000000 00000000 00000001 00000011 00000000 00000000
// \______________>>14_______________/ \______________>>14_______________/ \______________>>14_______________/ \______________>>14_______________/
//              00001111______                      00001011______                      00001100______                            00001100______
//                            |_______                            |_______                            |_______                            |_______
// 00000001 00000011 00000001 00001111 00000001 00000010 00000001 00001011 00000001 00000011 00000000 00001100 00000001 00000011 00000000 00001100
// \________________________________>>28_________________________________/ \________________________________>>28_________________________________/
//                                11111011________________________                                        11001100________________________
//                                                                |_______                                                                |_______
// 00000001 00000011 00000001 00001111 00000001 00000010 00000001 11111011 00000001 00000011 00000000 00001100 00000001 00000011 00000000 11001100
// <<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<< REVERSE (vrbitq_u8) >>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>
// 10000000 11000000 10000000 11110000 10000000 01000000 10000000 11011111 10000000 11000000 00000000 00110000 10000000 11000000 00000000 00110011
// \__00__/ \__01__/ \__02__/ \__03__/ \__04__/ \__05__/ \__06__/ \__07__/ \__08__/ \__09__/ \__10__/ \__11__/ \__12__/ \__13__/ \__14__/ \__15__/
//                                                                |      |________________________________________________________________|      |
//                                                                |______________________________         |         _____________________________|
//                                                                                               |11011111|00110011|
//                                                                                               |  LOW   |  HIGH  |
````