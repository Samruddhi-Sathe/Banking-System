# Banking-System
from __future__ import annotations
from abc import ABC, abstractmethod
from dataclasses import dataclass
from datetime import datetime
from typing import Dict, List, Optional, Iterable
from uuid import uuid4
import hashlib


# ============ Utilities ============

def _hash_password(raw: str) -> str:
    return hashlib.sha256(raw.encode("utf-8")).hexdigest()


# ============ Customer ============

class Customer:
    """
    Encapsulation: password hash is private; verify via method.
    """
    def __init__(self, customer_id: str, name: str, email: str, password: str):
        self.id = customer_id
        self.name = name
        self.email = email
        self._password_hash = _hash_password(password)
        self._account_numbers: List[str] = []  # owned account numbers

    def verify_password(self, raw: str) -> bool:
        return self._password_hash == _hash_password(raw)

    def _link_account(self, acct_no: str) -> None:
        if acct_no not in self._account_numbers:
            self._account_numbers.append(acct_no)

    def _unlink_account(self, acct_no: str) -> None:
        if acct_no in self._account_numbers:
            self._account_numbers.remove(acct_no)

    def accounts(self) -> Iterable[str]:
        return list(self._account_numbers)

    def __repr__(self) -> str:
        return f"Customer(id={self.id!r}, name={self.name!r})"


# ============ Accounts (Abstraction + Inheritance) ============

class Account(ABC):
    """
    Abstract base account.
    Encapsulation: _balance and _acct_no are protected.
    Class attr: _acct_counter to help generate numbers.
    """
    _acct_counter: int = 10_000_000

    def __init__(self, owner: Customer, initial_balance: float = 0.0):
        if initial_balance < 0:
            raise ValueError("Initial balance cannot be negative")
        type_code = self.__class__.__name__[0].upper()  # S for Savings, C for Checking
        Account._acct_counter += 1
        self._acct_no: str = f"{type_code}{Account._acct_counter}"
        self._balance: float = float(initial_balance)
        self.owner: Customer = owner
        self.created_on: datetime = datetime.now()
        self._txn_history: List["Transaction"] = []

    # ----- Properties (read-only acct_no, safe balance getter) -----
    @property
    def acct_no(self) -> str:
        return self._acct_no

    def get_balance(self) -> float:
        return round(self._balance, 2)

    # ----- Core behaviors -----
    def deposit(self, amount: float) -> None:
        if amount <= 0:
            raise ValueError("Deposit amount must be positive")
        self._balance += amount

    @abstractmethod
    def withdraw(self, amount: float) -> None:
        """Must be overridden to implement account-specific rules."""

    # ----- Helpers -----
    def _record(self, txn: "Transaction") -> None:
        self._txn_history.append(txn)

    def history(self) -> Iterable["Transaction"]:
        return list(self._txn_history)

    # ----- Static/class helpers -----
    @staticmethod
    def is_valid_acct(acct_no: str) -> bool:
        # Example: S10000001 or C10000002
        return len(acct_no) >= 8 and acct_no[0] in {"S", "C"} and acct_no[1:].isdigit()


class SavingsAccount(Account):
    def __init__(self, owner: Customer, initial_balance: float = 0.0,
                 min_balance: float = 1000.0, interest_rate: float = 0.04):
        super().__init__(owner, initial_balance)
        self.min_balance = float(min_balance)
        self.interest_rate = float(interest_rate)

    def withdraw(self, amount: float) -> None:
        if amount <= 0:
            raise ValueError("Withdrawal amount must be positive")
        projected = self._balance - amount
        if projected < self.min_balance:
            raise ValueError("Cannot violate minimum balance requirement")
        self._balance -= amount

    def apply_interest(self) -> float:
        """Apply monthly interest; return interest added."""
        monthly_rate = self.interest_rate / 12.0
        interest = self._balance * monthly_rate
        self._balance += interest
        return round(interest, 2)


class CheckingAccount(Account):
    def __init__(self, owner: Customer, initial_balance: float = 0.0,
                 overdraft_limit: float = 5000.0):
        super().__init__(owner, initial_balance)
        self.overdraft_limit = float(overdraft_limit)

    def withdraw(self, amount: float) -> None:
        if amount <= 0:
            raise ValueError("Withdrawal amount must be positive")
        if self._balance - amount < -self.overdraft_limit:
            raise ValueError("Overdraft limit exceeded")
        self._balance -= amount


# ============ Transactions (Abstraction + Polymorphism) ============

@dataclass
class TxnResult:
    success: bool
    message: str


class Transaction(ABC):
    """
    Abstract transaction with execute/rollback contract.
    """
    def __init__(self, description: str = ""):
        self.id: str = str(uuid4())
        self.timestamp: datetime = datetime.now()
        self.status: str = "PENDING"  # PENDING | DONE | REVERSED | FAILED
        self.description: str = description

    @abstractmethod
    def execute(self) -> TxnResult: ...

    @abstractmethod
    def rollback(self) -> TxnResult: ...

    def __repr__(self) -> str:
        return f"<Txn {self.__class__.__name__} id={self.id[:8]} status={self.status}>"


class DepositTxn(Transaction):
    def __init__(self, account: Account, amount: float):
        super().__init__(f"Deposit {amount} to {account.acct_no}")
        self.account = account
        self.amount = float(amount)

    def execute(self) -> TxnResult:
        try:
            self.account.deposit(self.amount)
            self.account._record(self)
            self.status = "DONE"
            return TxnResult(True, "Deposit successful")
        except Exception as e:
            self.status = "FAILED"
            return TxnResult(False, f"Deposit failed: {e}")

    def rollback(self) -> TxnResult:
        if self.status != "DONE":
            return TxnResult(False, "Cannot rollback; transaction not completed")
        try:
            self.account.withdraw(self.amount)  # may raise if rules prohibit
            self.status = "REVERSED"
            return TxnResult(True, "Deposit reversed")
        except Exception as e:
            return TxnResult(False, f"Rollback failed: {e}")


class WithdrawTxn(Transaction):
    def __init__(self, account: Account, amount: float):
        super().__init__(f"Withdraw {amount} from {account.acct_no}")
        self.account = account
        self.amount = float(amount)

    def execute(self) -> TxnResult:
        try:
            self.account.withdraw(self.amount)
            self.account._record(self)
            self.status = "DONE"
            return TxnResult(True, "Withdrawal successful")
        except Exception as e:
            self.status = "FAILED"
            return TxnResult(False, f"Withdrawal failed: {e}")

    def rollback(self) -> TxnResult:
        if self.status != "DONE":
            return TxnResult(False, "Cannot rollback; transaction not completed")
        try:
            self.account.deposit(self.amount)
            self.status = "REVERSED"
            return TxnResult(True, "Withdrawal reversed")
        except Exception as e:
            return TxnResult(False, f"Rollback failed: {e}")


class TransferTxn(Transaction):
    def __init__(self, from_acct: Account, to_acct: Account, amount: float):
        desc = f"Transfer {amount} from {from_acct.acct_no} to {to_acct.acct_no}"
        super().__init__(desc)
        self.from_acct = from_acct
        self.to_acct = to_acct
        self.amount = float(amount)
        self._completed_withdraw = False

    def execute(self) -> TxnResult:
        try:
            self.from_acct.withdraw(self.amount)
            self._completed_withdraw = True
            self.to_acct.deposit(self.amount)
            self.from_acct._record(self)
            self.to_acct._record(self)
            self.status = "DONE"
            return TxnResult(True, "Transfer successful")
        except Exception as e:
            # If withdraw succeeded but deposit failed, attempt to revert withdraw
            if self._completed_withdraw:
                try:
                    self.from_acct.deposit(self.amount)
                except Exception:
                    pass
            self.status = "FAILED"
            return TxnResult(False, f"Transfer failed: {e}")

    def rollback(self) -> TxnResult:
        if self.status != "DONE":
            return TxnResult(False, "Cannot rollback; transaction not completed")
        try:
            self.to_acct.withdraw(self.amount)
            self.from_acct.deposit(self.amount)
            self.status = "REVERSED"
            return TxnResult(True, "Transfer reversed")
        except Exception as e:
            return TxnResult(False, f"Rollback failed: {e}")


# ============ Bank (Aggregator/Owner) ============

class Bank:
    """
    Owns customers and accounts.
    Shows classmethod & staticmethod usage.
    """
    _bank_registry: Dict[str, "Bank"] = {}

    def __init__(self, name: str):
        self.name = name
        self._customers: Dict[str, Customer] = {}
        self._accounts: Dict[str, Account] = {}
        Bank._bank_registry[name] = self

    # ---- Customer management ----
    def add_customer(self, customer: Customer) -> None:
        if customer.id in self._customers:
            raise ValueError("Customer already exists")
        self._customers[customer.id] = customer

    def get_customer(self, customer_id: str) -> Customer:
        return self._customers[customer_id]

    # ---- Account lifecycle ----
    def open_account(self, customer_id: str, account_type: str,
                     initial_deposit: float = 0.0, **kwargs) -> Account:
        customer = self.get_customer(customer_id)

        if account_type.lower() in ("savings", "s"):
            acct: Account = SavingsAccount(customer, initial_deposit,
                                           kwargs.get("min_balance", 1000.0),
                                           kwargs.get("interest_rate", 0.04))
        elif account_type.lower() in ("checking", "c", "current"):
            acct = CheckingAccount(customer, initial_deposit,
                                   kwargs.get("overdraft_limit", 5000.0))
        else:
            raise ValueError("Unknown account type")

        # Register
        self._accounts[acct.acct_no] = acct
        customer._link_account(acct.acct_no)
        return acct

    def close_account(self, acct_no: str) -> None:
        acct = self.find_account(acct_no)
        if acct.get_balance() != 0:
            raise ValueError("Cannot close account with non-zero balance")
        owner = acct.owner
        owner._unlink_account(acct_no)
        del self._accounts[acct_no]

    def find_account(self, acct_no: str) -> Account:
        if acct_no not in self._accounts:
            raise KeyError("Account not found")
        return self._accounts[acct_no]

    def apply_interest(self) -> float:
        """
        Apply monthly interest to all savings accounts.
        Returns total interest paid.
        """
        total = 0.0
        for acct in self._accounts.values():
            if isinstance(acct, SavingsAccount):
                total += acct.apply_interest()
        return round(total, 2)

    def accounts_of(self, customer_id: str) -> List[Account]:
        cust = self.get_customer(customer_id)
        return [self._accounts[a] for a in cust.accounts()]

    # ---- Class/static demo ----
    @classmethod
    def all_banks(cls) -> List[str]:
        return list(cls._bank_registry.keys())

    @staticmethod
    def is_valid_acct(acct_no: str) -> bool:
        return Account.is_valid_acct(acct_no)

    def total_accounts(self) -> int:
        return len(self._accounts)

    def __repr__(self) -> str:
        return f"<Bank {self.name} customers={len(self._customers)} accounts={len(self._accounts)}>"


# ============ Demo ============

if __name__ == "__main__":
    bank = Bank("NeetBank")

    # Add customers
    alice = Customer("C001", "Alice", "alice@example.com", "alice@123")
    bob   = Customer("C002", "Bob",   "bob@example.com",   "bob@123")
    bank.add_customer(alice)
    bank.add_customer(bob)

    # Open accounts
    alice_sav = bank.open_account("C001", "savings", initial_deposit=5000, min_balance=1000, interest_rate=0.06)
    bob_chk   = bank.open_account("C002", "checking", initial_deposit=2000, overdraft_limit=3000)

    print(bank)  # quick snapshot
    print("Alice savings:", alice_sav.acct_no, alice_sav.get_balance())
    print("Bob checking :", bob_chk.acct_no, bob_chk.get_balance())

    # Transactions
    print("\n-- Transactions --")
    t1 = DepositTxn(alice_sav, 1200)
    print("Deposit:", t1.execute(), "Balance:", alice_sav.get_balance())

    t2 = WithdrawTxn(bob_chk, 3500)  # will use overdraft if needed
    print("Withdraw:", t2.execute(), "Balance:", bob_chk.get_balance())

    t3 = TransferTxn(alice_sav, bob_chk, 2000)
    print("Transfer:", t3.execute(), "Alice:", alice_sav.get_balance(), "Bob:", bob_chk.get_balance())

    # Apply interest to all savings
    paid = bank.apply_interest()
    print("\nInterest paid this month:", paid)
    print("Alice after interest:", alice_sav.get_balance())

    # Rollback example (optional)
    print("\n-- Rollback last transfer --")
    print("Rollback:", t3.rollback(), "Alice:", alice_sav.get_balance(), "Bob:", bob_chk.get_balance())

    # Histories
    print("\nAlice Txns:", list(alice_sav.history()))
    print("Bob   Txns:", list(bob_chk.history()))
