const User = require("../../models/User");
const bcrypt = require("bcrypt");
const { validationResult } = require("express-validator");
const jwt = require("jsonwebtoken");
const CustomError = require("../../utils/errors/CustomError");

//User login
const userLogin = async (req, res, next) => {
  const { email, password } = req.body;
  try {
    const user = await User.findOne({ email: email }).select([
      "password",
      "attempts",
      "lockeTime",
    ]);
    if (!user) {
      return next(new CustomError("User Not Register", 404));
    }
    const { lockeTime, attempts } = user;
    console.log("Lock User", user);
    if (lockeTime > Date.now()) {
      const remaningTime = Math.ceil((lockeTime - new Date()) / 60000);
      return next(
        new CustomError(
          `Your Account Has been Lock. Please try again after ${remaningTime} minutes.`,
          403
        )
      );
    }

    const match = bcrypt.compareSync(password, user.password);
    if (!match) {
      //Lock user for 1 hour after 3 unsuccessfull login attempts with wrong password
      user.attempts += 1;
      await user.save();
      if (attempts >= 2) {
        const lockeTime = new Date();
        lockeTime.setHours(lockeTime.getHours() + 1);
        await User.updateOne(
          { email },
          { $set: { attempts: 0 }, lockeTime: lockeTime }
        );ƒ
        return next(new CustomError("your Account lock for 1 hr", 401));
      }
      return next(new CustomError("Email Or Password doesn't match", 401));
    } else {
      user.attempts = 0;
      user.lockeTime = null;
      await user.save();
      const token = jwt.sign({ _id: user._id }, process.env.JWT_SECRET_KEY, {
        expiresIn: "20m",
      });
      return res.status(200).json({
        status: "success",
        message: "User Login Success",
        token: token,
      });
    }
  } catch (error) {
    console.log(error);
    return next(new CustomError("Unable to Login", 500));
  }
};

module.exports = userLogin;
