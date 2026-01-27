# Hawk High - Findings Report

# Table of contents
- ## [Contest Summary](#contest-summary)
- ## [Results Summary](#results-summary)
- ## High Risk Findings
    - ### [H-01. Wrong calculations for payPerTeacher in graduateAndUpgrade](#H-01)

- ## Low Risk Findings
    - ### [L-01. Principal can be added as Teacher](#L-01)


# <a id='contest-summary'></a>Contest Summary

### Sponsor: First Flight #39

### Dates: May 1st, 2025 - May 8th, 2025

[See more contest details here](https://codehawks.cyfrin.io/c/2025-05-hawk-high)

# <a id='results-summary'></a>Results Summary

### Number of findings:
- High: 1
- Medium: 0
- Low: 1


# High Risk Findings

## <a id='H-01'></a>H-01. Wrong calculations for payPerTeacher in graduateAndUpgrade            



## Summary

Wrong calculations for teachers payment

## Vulnerability Details

Each teacher should receive a share from the 35% of the bursary.

At the moment each teacher will receive 35% of the bursary.
The calculations here are incorrect.
[GitHub Link: LevelOne.sol](https://github.com/CodeHawks-Contests/2025-05-hawk-high/blob/main/src/LevelOne.sol#L302)

Proof of Concept:

```Solidity
 function graduateAndUpgrade(address _levelTwo, bytes memory) public onlyPrincipal {
        if (_levelTwo == address(0)) {
            revert HH__ZeroAddress();
        }

        uint256 totalTeachers = listOfTeachers.length;

        uint256 payPerTeacher = (bursary * TEACHER_WAGE) / PRECISION; // The wrong calculations are located here
        uint256 principalPay = (bursary * PRINCIPAL_WAGE) / PRECISION;

        _authorizeUpgrade(_levelTwo);

        for (uint256 n = 0; n < totalTeachers; n++) {
            usdc.safeTransfer(listOfTeachers[n], payPerTeacher);
        }

        usdc.safeTransfer(principal, principalPay);
    }
```

Proving this is easy - adding one more teacher leads to:

`[FAIL: ERC20InsufficientBalance(0x90193C961A926261B756D1E5bb255e67ff9498A1, 9000000000000000000000 [9e21], 10500000000000000000000 [1.05e22])] test_confirm_can_graduate() `  &#x20;

```Solidity
  function _teachersAdded() internal {
        vm.startPrank(principal);
        levelOneProxy.addTeacher(alice);
        levelOneProxy.addTeacher(bob);
        levelOneProxy.addTeacher(makeAddr("teacher_3"));
        vm.stopPrank();
    }
```

At: [GitHub Link: LeveOnelAndGraduateTest.t.sol](https://github.com/CodeHawks-Contests/2025-05-hawk-high/blob/main/test/LeveOnelAndGraduateTest.t.sol#L187)

There is no more funds, as each of the 3 Teachers is going to receive 35%

## Impact

The rewards for teachers can cause DoS or overpaying for the teachers if they are only 2.

## Tools Used

Manual Review

## Recommendations

* Change the logic for teachers rewards.

For example:

```Diff
-uint256 payPerTeacher = (bursary * TEACHER_WAGE) / PRECISION;
+uint256 totalPayForTeachers = (bursary * TEACHER_WAGE) / PRECISION;
+uint256 payPerTeacher = totalPayForTeachers / totalTeachers;
```

    


# Low Risk Findings

## <a id='L-01'></a>L-01. Principal can be added as Teacher            



## Summary

There is no check in `addTeacher` function if input address is currently the address of the Principal.

## Vulnerability Details

Adding Principal as Teacher causes the principal to receive his 5% and payment for Teacher.

Proof of Concept:

```Solidity
 function _teachersAdded() internal {
        vm.startPrank(principal);
        levelOneProxy.addTeacher(principal); // Adding the Principal as Teacher address by accident or no
        levelOneProxy.addTeacher(bob);
        vm.stopPrank();
    }
```

[GitHub Link: LeveOnelAndGraduateTest.t.sol](https://github.com/CodeHawks-Contests/2025-05-hawk-high/blob/main/test/LeveOnelAndGraduateTest.t.sol#L189)

As we can see after adding the Principal as a Teacher and run the test
we observe that the Principal receives his part of the payment + the part for a teacher.

![tests](https://i.imgur.com/ZP2hSKU.png)

## Impact

* Also this gives him full rights as the Teachers.
* Malicious Principal (if it's possible for Principal to be malicious) can maximise his rewards by using his advantage to be the both roles at the same time
  and remove part of the teachers or all except him.

## Tools Used

Manual Review

## Recommendations

Adding a check in `addTeacher` solves the issue - the Principal address prevented from participating as Teacher.

[GitHub Link: LevelOne.sol](https://github.com/CodeHawks-Contests/2025-05-hawk-high/blob/main/src/LevelOne.sol#L201)

```diff
  function addTeacher(address _teacher) public onlyPrincipal notYetInSession {
        if (_teacher == address(0)) {
            revert HH__ZeroAddress();
        }

        if (isTeacher[_teacher]) {
            revert HH__TeacherExists();
        }

        if (isStudent[_teacher]) {
            revert HH__NotAllowed();
        }

+       if (principal == _teacher) {
+           revert HH__NotAllowed();
+       }

        listOfTeachers.push(_teacher);
        isTeacher[_teacher] = true;

        emit TeacherAdded(_teacher);
    }
```



