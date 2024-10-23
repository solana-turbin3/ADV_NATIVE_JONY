# ADV_NATIVE_JONY_PREREQS

### Entrypoint

## PaulX- What Lines? 
```rust
entrypoint!(process_instruction);
```

## Rust Concept? 

- Macros, Paulx uses the entrypoint!(process_instruction) macro to register the process_instruction as the program's entry point.
## Optimization Ideas

## Notes

The entrypoint.rs should only handle dispatching calls to other processor functions based on the received instruction.

```rust
fn process_instruction(
    program_id: &Pubkey,
    accounts: &[AccountInfo],
    instruction_data: &[u8],
) -> ProgramResult
```

### IX Discriminator

## PaulX- What Lines? 
```rust

 pub enum EscrowInstruction {

    InitEscrow {
        /// The amount party A expects to receive of token Y
        amount: u64
    }
}

...

let (tag, rest) = input.split_first().ok_or(InvalidInstruction)?;

```

## Rust Concept? 

- Enums, uses an enum to define program's various ix's.
- Pattern Matching, checks and extracts the first byte to determine which instruction is being executed. It uses pattern matching to map this "discriminator" byte to the correct variant of the EscrowInstruction enum.

## Optimization Ideas

## Notes

- this byte essentially serves as an identifier for each instruction,

### IX Serde

## PaulX- What Lines? 
```rust

impl EscrowInstruction {
    /// Unpacks a byte buffer into a [EscrowInstruction](enum.EscrowInstruction.html).
    pub fn unpack(input: &[u8]) -> Result<Self, ProgramError> {
        let (tag, rest) = input.split_first().ok_or(InvalidInstruction)?;

        Ok(match tag {
            0 => Self::InitEscrow {
                amount: Self::unpack_amount(rest)?,
            },
            _ => return Err(InvalidInstruction.into()),
        })
    }

    fn unpack_amount(input: &[u8]) -> Result<u64, ProgramError> {
        let amount = input
            .get(..8)
            .and_then(|slice| slice.try_into().ok())
            .map(u64::from_le_bytes)
            .ok_or(InvalidInstruction)?;
        Ok(amount)
    }
}

```

## Rust Concept? 

- Serde (Serialization/Deserialization): the ix deserialization is implemented manually in the unpack method.


## Optimization Ideas

## Notes

- Leverages split_first and try_into for direct and efficient byte manipulation.

### Account Serde

## PaulX- What Lines? 
```rust
impl Pack for Escrow {
    const LEN: usize = 105;
    fn unpack_from_slice(src: &[u8]) -> Result<Self, ProgramError> {
        let src = array_ref![src, 0, Escrow::LEN];
        let (
            is_initialized,
            initializer_pubkey,
            temp_token_account_pubkey,
            initializer_token_to_receive_account_pubkey,
            expected_amount,
        ) = array_refs![src, 1, 32, 32, 32, 8];
        let is_initialized = match is_initialized {
            [0] => false,
            [1] => true,
            _ => return Err(ProgramError::InvalidAccountData),
        };

        Ok(Escrow {
            is_initialized,
            initializer_pubkey: Pubkey::new_from_array(*initializer_pubkey),
            temp_token_account_pubkey: Pubkey::new_from_array(*temp_token_account_pubkey),
            initializer_token_to_receive_account_pubkey: Pubkey::new_from_array(*initializer_token_to_receive_account_pubkey),
            expected_amount: u64::from_le_bytes(*expected_amount),
        })
    }
}
```

## Rust Concept? 

- Traits implementations: Pack and Sealed.
- Bytee Slicing, using the arrayref! and array_mut_ref! macros to access specific sections of the byte array.


## Optimization Ideas

## Notes

- The pack trait wich byte slicing is well suited for high-performance nee4ds.


### Account Checks
## PaulX- What Lines? 

inside process_init_escrow

- Checking that accounts are owned by the expected program.
```rust
*token_to_receive_account.owner != spl_token::id()
```
- Comparing account public keys to match expected values.
```rust
escrow_info.initializer_pubkey != *initializers_main_account.key
```

## Rust Concept? 

- Dereferencing and Ownership Checks.
- Public Key Comparisons
## Optimization Ideas

## Notes

These account checks are essential to ensure integrity within Solana programs.


### Balance Checks

Inside the process_exchange 
## PaulX- What Lines? 
```rust
if amount_expected_by_taker != pdas_temp_token_account_info.amount {
    return Err(EscrowError::ExpectedAmountMismatch.into());
}
```

## Rust Concept? 

- Basic comparison and error handling.

## Notes

- This comparison is crucial to prevent mismatches in the escrow amounts, ensuring the expected value is transferred.


### CPIs

## PaulX- What Lines? 
inside process_init_escrow
```rust
let owner_change_ix = spl_token::instruction::set_authority(
    token_program.key,
    temp_token_account.key,
    Some(&pda),
    spl_token::instruction::AuthorityType::AccountOwner,
    initializer.key,
    &[&initializer.key],
)?;

msg!("Calling the token program to transfer token account ownership...");
invoke(
    &owner_change_ix,
    &[
        temp_token_account.clone(),
        initializer.clone(),
        token_program.clone(),
    ],
)?;

```

## Rust Concept? 

- CPI calls with invoke.


The entrypoint.rs should only handle dispatching calls to other processor functions based on the received instruction.


### Signer Checks

## PaulX- What Lines? 

In both process_init_escrow and process_exchange, signer checks are done using:
```rust
if !initializer.is_signer { ... }
if !taker.is_signer { ... }
```

## Rust Concept? 
Signer Verification: Directly using .is_signer to verify that a required account has signed the transaction.
## Optimization Ideas

## Notes

These ensure that actions requiring authorization have the necessary signatures.


### PDAs
## PaulX- What Lines? 
The creation and use of a PDA inside process_exchange
```rust
let (pda, bump_seed) = Pubkey::find_program_address(&[b"escrow"], program_id);
```

## Rust Concept? 

- Finding PDAs: The use of find_program_address to derive a deterministic, program-specific address using seeds and a bump value.
## Optimization Ideas

- Reuse the saved bump.

## Notes

When PDAs are involved in signing, we use invoke_signed with the derived address and seeds.



### Error Handling & Testing

## PaulX- What Lines? 
```rust
#[derive(Error, Debug, Copy, Clone)]
pub enum EscrowError {
    /// Invalid instruction
    #[error("Invalid Instruction")]
    InvalidInstruction,
}

impl From<EscrowError> for ProgramError {
    fn from(e: EscrowError) -> Self {
        ProgramError::Custom(e as u32)
    }
}
```
Throughout the code we handle the errors:
```rust
.ok_or(ProgramError::InvalidAccountData)
```

```rust
.ok_or(EscrowError::AmountOverflow)?
```
## Rust Concept? 

- Custom Errors: the use of a custom error enum EscrowError
## Optimization Ideas

## Notes

The use of custom error enums with clear messages helps in debugging and testing. 