# login

const User = require("../../models/User");
const bcrypt = require("bcrypt");
const { validationResult } = require("express-validator");
const jwt = require("jsonwebtoken");
const CustomError = require("../../utils/errors/CustomError");

//User login
const userLogin = async (req, res, next) => {
  const { email, password } = req.body;
  try {

  const lockUser = await User.findOne({ email, lockeTime: { $gt: new Date() } });
  console.log("Lock User", lockUser);
  if (lockUser) {
     const leftTime = Math.ceil((lockUser.lockeTime - new Date()) / 60000); 
     console.log("Left time", leftTime);
    return next(
      new CustomError(
        `Your Account Has been Lock. Please try again after ${leftTime} minutes.`,
        403
      )
    );
   }

    const user = await User.findOne({ email: email })
    if (!user) {
      return next(new CustomError("User Not Register", 404));
    }

    const match = bcrypt.compareSync(password, user.password);
    if (!match) {
      //Lock user for 1 hour after 3 unsuccessfull login attempts with wrong password
      await User.updateOne({ email }, { $inc: { attempts: 1 } });
      const attempt = await User.findOne({ email });
      if (attempt && attempt.attempts >= 3) {
        const lockeTime = new Date();
        lockeTime.setHours(lockeTime.getHours() + 1);
        await User.updateOne({ email }, { $set: { attempts: 0 }, lockeTime: lockeTime });
        return next(
          new CustomError(
            "your Account lock for 1 hr",
            401
          )
        );
      }
      return next(new CustomError("Email Or Password doesn't match", 401));
    }
    const token = jwt.sign({ _id: user._id }, process.env.JWT_SECRET_KEY, {
      expiresIn: "20m",
    });
    return res
      .status(200)
      .json({ status: "success", message: "User Login Success", token: token });
  } catch (error) {
    console.log(error);
    return next(new CustomError("Unable to Login", 500));
  }
};

module.exports = userLogin;
